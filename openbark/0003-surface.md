# RFC 0003: Surface

- **Status:** Draft
- **Author(s):** @qianxuege
- **Created:** 2026-10-03
- **Updated:** 2026-10-03
- **Affects:** openbark-surface

## Overview

This RFC defines the design of `openbark-surface`, the member-facing half of OpenBark: a React single-page app and an Express backend-for-frontend. The Surface signs members in with Andrew ID, resolves their roles, authorizes every request, stores chats and their citations, holds the approval queue and the audit log, and relays answers from the Agent. It follows bark RFC 0003 closely and departs from it in authorization, the signed user assertion, chat visibility, and the approval and audit surfaces.

## Motivation

The Surface holds OpenBark's only authentication boundary and its only record of who did what. Two things make it more than Bark's Surface.

OpenBark authorizes rather than only authenticating. Bark has one authorization decision: whether a member is in `ADMIN_GROUP`. OpenBark decides which corpora a member can retrieve from and which actions they can approve, and those decisions have to be made server-side, consistently, in one place.

OpenBark also has to keep a record. When a broadcast goes out or a reservation is made, the question "who approved this, based on what, and when" needs an answer that is not a log line. The approval queue and the audit log are first-class parts of the Surface, not instrumentation.

## Goals

- Define authentication, sessions, and the identity provider interface
- Define the role and permission model, and how it is sourced from governance without being welded to it
- Define the signed user assertion sent to the Agent
- Define chat visibility and what sharing means when answers are role-filtered
- Define the approval queue, the audit log, and their APIs
- Define what the Surface stores

## Non-Goals

- Visual design and UI copy
- Agent behavior (RFC 0002), tool authoring (RFC 0004), retrieval (RFC 0005), and the action contract (RFC 0006)
- Provisioning Keycloak groups, which governance owns

## Detailed Design

### Stack

A Deno workspace with two apps, matching `cmugpt-surface` closely enough that the two share patterns and contributors:

| App | Stack |
|---|---|
| `apps/web` | React 19, Vite, TanStack Router and TanStack Query, Tailwind CSS v4 |
| `apps/server` | Express 5, tsoa, Zod, Drizzle ORM on PostgreSQL |

### Type-safe API

Unchanged from bark RFC 0003: `apps/server` is the source of truth, tsoa generates the OpenAPI spec and Express routes from its controllers, `apps/web` consumes the generated types, and streaming and auth routes are hand-written because they do not fit tsoa's model. Kept for Bark's reason, that a server change becomes a compile error on the frontend.

### Authentication and sessions

Unchanged from bark RFC 0003, which this RFC does not restate: OIDC against ScottyLabs' Keycloak with PKCE and a nonce, pending logins in `oidc_login_states` expiring after 10 minutes, server-side sessions with an opaque id in an httpOnly cookie and Keycloak's tokens never reaching the frontend, and redirects through the shared ricochet relay so preview deployments sign in without registering redirect URIs. Locally devenv runs ricochet on `127.0.0.1:8091`.

One change: there is no unauthenticated surface at all. Bark serves public shared chats to anyone with a link. Every OpenBark route except `/api/health` and the auth routes requires a session, which is also what makes the visibility model below coherent.

### Identity provider interface

Roles do not come from OpenBark. The Surface resolves them through one module with a narrow interface, so the source can be swapped:

```ts
interface IdentityProvider {
  // Role ids for a subject, called at sign-in and on re-check.
  resolveRoles(subject: string): Promise<RoleId[]>;
  // Opaque version that changes when any role assignment changes.
  rolesVersion(): Promise<string>;
}
```

The reference implementation reads Keycloak group memberships from the ID token's claims, with group-to-role mapping in configuration. Groups are provisioned from the `scottylabs` organization and team files in governance, so governance remains the place a person's membership is edited, and OpenBark learns about it through Keycloak.

Nothing above the interface knows about Keycloak or governance. Replacing it with a different directory, or with a ScottyLabs authorization service if one appears, means one module and the group-to-role map.

### Roles and permissions

- **Role ids** are opaque strings from the identity provider, for example `slai`, `slai-lead`, `devops`, `member`. Every signed-in member of the organization has `member`.
- **Permissions** are OpenBark's own vocabulary and are defined in `apps/server/src/lib/permissions.ts`. They name a capability, not a person: `kb.read.<corpus>`, `action.preview.<class>`, `action.approve.<class>`, `audit.read`, `admin`.
- **The mapping** from role ids to permissions is data, reviewed in pull requests. A new corpus or action class adds permissions and maps them; it does not add a code path.
- **Resolution** happens at sign-in and is cached on the session with the `rolesVersion` it was computed from. A write request re-resolves when the version has changed. A read request uses the cached set, which can be stale for at most the session's re-check interval, default 15 minutes.
- **Every authorization decision is server-side.** The frontend hides what a member cannot do, and the server rejects it regardless.

Bark's single `ADMIN_GROUP` check becomes `admin` in this model, so the existing pattern survives as a special case.

### Signed user assertion

The Surface sends the Agent a short-lived signed assertion rather than Bark's hashed subject:

```json
{
  "sub": "oidc:<subject>",
  "roles": ["member", "slai"],
  "permissions": ["kb.read.rfcs", "action.preview.record"],
  "iss": "openbark-surface",
  "aud": "openbark-agent",
  "exp": "<now + 120s>",
  "jti": "<unique>"
}
```

It is signed with a key the Agent verifies (`ASSERTION_PUBLIC_KEY`, RFC 0002) and lives for two minutes, long enough for one turn. Email, display name, and the Keycloak tokens are not in it.

This reverses bark RFC 0003's `agentUserId(sub)` hashing. That decision kept the raw subject from leaving the Surface, which was free for Bark because the Agent needed identity only to partition memory. OpenBark's Agent filters retrieval by role, binds write tools by permission, and records executions against a person, and a hash supports none of those. The assertion recovers what the hash was protecting in a different way: it is short-lived, audience-bound, carries no contact information, and is verifiable, so a leaked one expires rather than being a durable identifier.

The Surface remains the only component that can map a subject to a person.

### Data

| Table | Contents |
|---|---|
| `auth_sessions`, `oidc_login_states` | Sessions and pending logins |
| `members` | Subject, cached role ids, `roles_version`, last sign-in |
| `chats` | One row per chat, owned by a subject, with a title, `starred`, and `visibility` |
| `chat_grants` | Explicit grants on a chat, to a subject or a role id |
| `messages` | Member and assistant messages, and the memory a turn saved |
| `message_citations` | Per-message citation ids with the corpus each came from |
| `actions` | Pending and resolved actions: class, provider, arguments, preview, idempotency key, run id, state, requester, approver, timestamps |
| `audit_log` | Append-only: actor, action, arguments, result, approver, timestamp |

Migrations live in `apps/server/drizzle` and run at startup. Chats cascade to messages, citations, and grants. The Surface stores no personal memory facts and no documents; those belong to the Agent and the index.

`message_citations` exists so a shared chat can be re-authorized against a viewer. Without it, sharing would require re-running the turn.

### Chat visibility and sharing

Bark has an `is_public` boolean that makes a chat readable by anyone with the link, signed in or not. That is wrong for an internal assistant, but the capability it provides is not: the main reason to share an OpenBark chat is that the answer is useful to the rest of the team, and an admin sharing a chat with other admins is exactly the intended use.

So the boolean becomes a scope, and no scope reaches the public internet:

| Visibility | Who can read |
|---|---|
| `private` | The owner only. The default. |
| `shared` | Subjects and role ids named in `chat_grants` |
| `org` | Any signed-in member of the organization |

A link is not an authorization. Opening a shared chat requires a session and a grant.

**Citations are re-authorized on view.** This is the part that matters. An answer was retrieved under the author's roles, so a chat can contain material the viewer has no permission to retrieve. When anyone other than the owner opens a chat, the Surface checks each `message_citations` row against the viewer's permissions and:

- renders citations the viewer can read as normal links,
- renders citations the viewer cannot read as a withheld marker naming the corpus but not the content,
- and marks the message as partially withheld so the viewer knows the answer rests on something they cannot see.

The answer text itself is not redacted, because an answer that quotes a restricted document in prose cannot be reliably scrubbed. Sharing a chat is therefore a disclosure the owner makes deliberately, and the UI says so at the moment of sharing: it lists which corpora the chat draws on, and warns when the chat cites a corpus narrower than the audience being granted. A chat citing a corpus the owner cannot share onward is restricted to `private` and `shared` with subjects who already hold the permission.

### Approvals

The approval queue is the Surface's half of the action contract (RFC 0006).

1. The Agent emits `action_preview` and then `action_pending` with a run id. The Surface stores a row in `actions` with state `pending`.
2. The member sees the preview inline in the chat. Members with `action.approve.<class>` see it in a queue at `/approvals`, whether or not they were in the conversation.
3. An approver approves or rejects. The Surface records the approver and timestamp, writes to `audit_log`, and calls the Agent's resume or abandon route.
4. A pending action expires after the Agent's TTL. The Surface marks it `expired` and stops showing it as actionable.

Rules the Surface enforces:

- **Four eyes where the class requires it.** An action class may be configured to require an approver other than the requester. Every irreversible class should be.
- **The preview is immutable.** An approver approves stored bytes. The Agent re-checks the arguments against them on resume (RFC 0002).
- **Approval is a permission, not a role.** `action.approve.broadcast` and `action.approve.record` are separate, so trusting someone to create a task does not trust them to email the organization.

### Audit

`audit_log` is append-only and has no update or delete route. It records every previewed action, approval, rejection, expiry, and execution with its arguments and result. Members with `audit.read` read it at `/audit`, filterable by actor, class, and date. A member can always see their own entries.

Write tool arguments are stored here, which departs from bark RFC 0004's rule that tool arguments are never logged. RFC 0006 defines the record's contents and argues the split. The Surface's part is that access is gated on `audit.read`, retention is finite (`AUDIT_RETENTION_DAYS`), and there is no route that mutates a row.

### API

| Route | Purpose |
|---|---|
| `GET /chats`, `POST /chats` | List and create chats |
| `GET /chats/{id}`, `PATCH /chats/{id}`, `DELETE /chats/{id}` | Read, rename, retitle, and delete |
| `PATCH /chats/{id}/visibility` | Set visibility and grants |
| `GET /chats/{id}/messages`, `POST /chats/{id}/messages` | History, and a message without streaming |
| `POST /chats/{id}/messages/stream` | Send a message and stream the answer |
| `GET /actions`, `GET /actions/{id}` | The approval queue and one action with its preview |
| `POST /actions/{id}/approve`, `POST /actions/{id}/reject` | Resolve a pending action |
| `GET /audit` | The audit log, gated on `audit.read` |
| `GET /me`, `GET /me/permissions` | The member and their resolved permissions |
| `GET /me/models`, `GET /me/preferences`, `PATCH /me/preferences` | Model list and preferred model |
| `GET /me/memories`, `DELETE /me/memories/{kind}/{id}`, `DELETE /me/memories` | Personal memory, proxied to the Agent |
| `GET /api/health` | Status |

Feature-specific routes are deliberately absent. An earlier draft of this design specified `GET /rooms` and `POST /emails/all`. Those are not Surface routes: room booking and mass emailing are action providers behind the contract in RFC 0006, reached through `/actions` like any other, so they get preview, approval, idempotency, and audit for free rather than each reimplementing them. A provider that also needs a non-chat UI gets a read route under `/providers/<id>/...` that reads through the same permission checks; it never gets its own write path.

### Chat flow

1. The web app posts to `POST /chats/{id}/messages/stream` with the message and the tool groups switched off.
2. The server checks that the member owns the chat, mints an assertion, stores the member message, and calls the Agent's stream route.
3. The server relays `status`, `delta`, `citation`, `action_preview`, `action_pending`, `action_result`, and `memory` events, and ignores types it does not recognize. It stops waiting after 90 seconds of silence.
4. On `citation`, the server records a `message_citations` row. On `action_pending`, it creates the `actions` row. On `done`, it stores the assistant message. A new chat gets a title from the Agent, or a shortened copy of the message if that call fails.

### Models and tool toggles

- **Models.** A curated list of ids a member can choose from, as in Bark, with the first as the default. Every entry must also be in the Agent's `ALLOWED_MODEL_IDS`; the Agent rejects anything outside it, so the Surface's list cannot widen the policy (RFC 0002).
- **Tool toggles.** A list of groups a member can switch off, matching the group table in RFC 0004. Sent with each message, not stored, with unknown ids dropped. These are a preference. A member cannot enable a tool their permissions do not cover, and the toggle list is not what prevents it.

### Configuration

Declared in `secretspec.toml` and validated at startup. Required: `OIDC_ISSUER_URL`, `OIDC_CLIENT_ID`, `OIDC_CLIENT_SECRET`, `ALLOWED_ORIGINS_REGEX`, `DATABASE_URL`, `AGENT_API_URL`, `AGENT_SHARED_SECRET`, `ASSERTION_PRIVATE_KEY`, and `ROLE_PERMISSION_MAP`. Optional: `OAUTH_RELAY_URL`, `APP_URL`, `ROLE_RECHECK_SECONDS` (default 900), `AUDIT_RETENTION_DAYS`. `AGENT_SHARED_SECRET` must be at least 32 characters in production.

### Testing

Unit tests for `apps/web` with Vitest and Testing Library, end-to-end tests with Playwright, and type checks for both apps, as in Bark. Two suites are required beyond that:

- **Authorization tests** covering every permission: a member without a permission is rejected on the server even with the frontend check bypassed.
- **Sharing tests** covering citation re-authorization: a viewer without a corpus permission sees the withheld marker, and the owner is warned when granting an audience wider than the chat's narrowest corpus.

### Deployment

Kennel serves `apps/web` as a static SPA and runs the API, compiled with `deno compile`. `flake.nix` defines both packages, as `cmugpt-surface`'s does (RFC 0001).

### Development environment

`devenv up` starts PostgreSQL, ricochet on 8091, the API on 3002, and Vite on 4174, with dev secrets from OpenBao. Without Nix, local sign-in is not available because ricochet comes from devenv.

## Alternatives Considered

- **Keeping `is_public` as Bark has it.** It is already built and sharing is a real need. Rejected only in its public form: an internal assistant must not serve retrieved institutional material to unauthenticated readers. The capability is preserved as `org` and `shared`, which covers the admin-sharing-with-admins case and more, and adds the citation re-authorization that a boolean cannot express.
- **Redacting the answer text for viewers who lack a citation's permission.** It would make sharing leak-proof. Rejected as not achievable: an answer paraphrases its sources in prose, and a scrubber that claims to remove restricted content from free text will be wrong sometimes, which is worse than a clear warning. Marking the message partially withheld and warning the owner at share time is honest about where the decision lies.
- **Keeping bark RFC 0003's hashed user id.** Covered above. It cannot express roles, cannot be mapped back to a person for an audit, and would push the authorization decision to the Surface for data the Surface does not hold.
- **Storing roles in OpenBark's own tables as the source of truth.** It removes the dependency on Keycloak and governance and allows roles governance does not model. Rejected: a second place to edit membership will drift from governance, and the team already maintains governance. The identity provider interface is the escape hatch if that changes.
- **Dedicated `GET /rooms` and `POST /emails/all` routes.** They were in the original feature list and are the obvious shape. Rejected because each would need its own preview, approval, idempotency, and audit handling, and the second one would copy the first. Behind the action contract, a provider declares a schema and gets all of it. `POST /emails/all` in particular is a route that sends mail to the organization on one call, and that should not exist at all.
- **Storing tool toggles server-side.** As in Bark, keeping them client-side avoids a schema change and they are a per-session choice. Unchanged, and safe here only because toggles are not a security control.

## Open Questions

- Should `org` visibility be available at all, or should the widest scope be `shared` with a role id? `org` is convenient and matches a ScottyLabs-wide audience, but it makes the warning at share time the only thing standing between a restricted citation and everyone.
- Is 15 minutes the right role re-check interval? A member removed from a team keeps read access for that long.
- Should the audit log be mirrored to an append-only external sink, so it survives loss of the Surface database (RFC 0001)?
- Does a non-chat provider UI need more than the read-only `/providers/<id>/...` shape proposed here? Deferring until a provider needs it.

## Implementation Phases

**Sign-in and authorization**

- OIDC, sessions, ricochet, `members`
- Identity provider interface with the Keycloak implementation, permissions vocabulary, role-to-permission map, server-side checks and tests
- Assertion minting

**Chat**

- `chats`, `messages`, `message_citations`, the stream relay, model list and preferences, personal memory proxy
- Visibility, grants, and citation re-authorization, with the sharing tests

**Approvals and audit**

- `actions`, the approval queue and inline previews, approve and reject, four-eyes configuration
- `audit_log` and `/audit`
