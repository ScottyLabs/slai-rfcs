# RFC 0004: MCP Tools

- **Status:** Draft
- **Author(s):** @qianxuege
- **Created:** 2026-10-03
- **Updated:** 2026-10-03
- **Affects:** openbark-knowledge, openbark-actions, openbark-agent, openbark-surface

## Overview

This RFC defines OpenBark's MCP tool servers and the tool contract the Agent and the Surface depend on: how tools are named and grouped, how a tool declares whether it reads or writes and what permission it needs, how the servers authenticate their callers, and what a module author does to add a group. It adopts bark RFC 0004's naming and group discipline and adds classification, authentication, and a split between read and write servers.

## Motivation

Bark's tool contract works and its central lesson is worth inheriting: a tool group is an id shared by several places, nothing enforces that they agree, and when the `bus` group shipped to only one of three lists the result was that five tool schemas were sent to the model on every turn and users could not switch them off. Nothing failed, which is why it went unnoticed. The group table and the registration checklist are the fix, and they carry over.

Two things do not carry over. Bark's MCP server is public, unauthenticated, stateless, and read-only, and bark RFC 0004 leans on all four properties, including to justify never logging tool arguments. OpenBark's tools read private material and change things outside OpenBark. A contract where the Agent cannot tell a search from a send is not usable here.

## Goals

- Define the read and write servers and why they are separate
- Define tool naming and groups, carried over from Bark
- Define tool classification: read or write, required permission, reversibility, idempotency
- Define how the servers authenticate the Agent and the member it acts for
- Define the process for adding a group, and keep the Surface, Agent, and server lists in agreement
- Set tool design guidelines, including provenance for citation

## Non-Goals

- How the Agent chooses among tools (RFC 0002)
- Retrieval behind the `kb` tools (RFC 0005)
- The approval lifecycle, which this RFC's classification feeds (RFC 0006)
- Any specific action provider

## Detailed Design

### Three servers, two postures

The Agent federates three MCP servers. Two are OpenBark's and one is Bark's.

| Server | Owner | Contains | Port | Posture |
|---|---|---|---|---|
| `openbark-knowledge` | OpenBark | Read tools over the index and live internal sources | 5053 | Authenticated, arguments not retained |
| `openbark-actions` | OpenBark | Action tools that change something | 5052 | Authenticated, arguments audited, approval required |
| `mcp-server` | Bark | Public campus data | 5050 | Public, foreign, read-only by configuration |

OpenBark's two are FastMCP servers serving streamable HTTP at `/mcp`, mounting a sub-server per group under its prefix, exactly as Bark's `main.py` does with `main_mcp.mount(eats_mcp, prefix="eats")`.

Read and write are separate services, and separate repositories (RFC 0001). Bark RFC 0004 rejects one server per group on the grounds that it multiplies deployments and Agent configuration for small, similar services. That reasoning holds for groups and is not what this is. The split here is by posture, not by group, and the two halves differ in every way that matters: whether arguments are retained, whether a permission is checked before binding, whether an approval gates execution, who can merge a change, and what the consequence of a bug is. In one process a misrouted mount makes a send look like a search. Separated, the dangerous half is small enough to read in full and narrow enough to restrict merge access to.

### Consuming a server OpenBark does not own

OpenBark answers campus questions with Bark's `mcp-server` rather than re-wrapping CMUEats, CMUMaps, CMUCourses, the guide, and the bus sign. Bark RFC 0001 predicted this when it kept the MCP server in its own repository because it "is useful beyond Bark." Making it work costs four rules, defined in RFC 0001 and implemented in the Agent:

1. **Group ids are globally unique across servers.** `MultiServerMCPClient.get_tools()` returns a flat list with no namespacing, and the Agent derives a group from the tool's `<group>_` prefix, so two servers publishing the same id would collide and a member's toggle for one would silence the other. The Agent refuses to start on a collision. OpenBark's `kb`, `gov`, and `forge` do not overlap Bark's `eats`, `maps`, `courses`, `guide`, and `bus`, and this check is what keeps that true as both grow.
2. **Classification comes from configuration for a foreign server.** Bark's tools carry none of the metadata below and should not have to. The Agent applies a per-server default, so every tool from `mcp-server` is `kind: read` with `permission: kb.read.public`. This is the property that lets OpenBark consume any well-behaved read-only MCP server without that server knowing OpenBark exists.
3. **The assertion is never forwarded to a foreign server.** It carries the member's subject, roles, and permissions, and Bark's server is public, unauthenticated, and operated by another project.
4. **A foreign server is a soft dependency.** Its outage removes its groups and is reported as degraded; it does not stop OpenBark answering.

A server may be configured as foreign-read only when the team trusts what it publishes, because the Agent will bind anything it finds there as a read tool. That trust is why this applies to another ScottyLabs service and not to an arbitrary third-party endpoint.

Nothing here asks Bark to change. The group table below therefore lists OpenBark's groups only; Bark's five are documented in bark RFC 0004 and are reached as a configured server, not registered here.

### Naming and groups

Carried over from bark RFC 0004 unchanged:

- Every tool is named `<group>_<action>` in snake_case. The prefix comes from the mount, so a sub-server names its tools just `<action>`.
- A group id is a short lowercase word matching `[a-z]+`, with no `_`, because the Agent finds a tool's group by its `<group>_` prefix.
- Every tool belongs to exactly one group. Ungrouped tools are not allowed: they cannot be switched off or narrowed, and they cost context on every turn.

Group table, the source of truth for ids OpenBark publishes. Bark's five remain bark RFC 0004's to own:

| Group id | Label | Server | Backed by | Classification |
|---|---|---|---|---|
| `kb` | Institutional Knowledge | read | `openbark-knowledge` (RFC 0005) | read |
| `forge` | git.cmu.dev | read | Forgejo API | read |
| `gov` | Governance | read | `openbark-knowledge`, governance files | read |

The write server starts empty. Its groups arrive with their providers (RFC 0006), one per capability class, and each lands in this table with the permission it requires.

Every row must appear in three places, as in Bark:

| Place | What |
|---|---|
| `openbark-knowledge` or `openbark-actions`, `src/.../main.py` | `main_mcp.mount(<sub>, prefix="<id>")` |
| `agent/src/openbark/mcp_tools.py` | An entry in the group labels and a hint pattern |
| `openbark-surface`, `apps/server/src/lib/agentTools.ts` and the chat UI | An entry in the toggle list and a toggle |

These are three repositories (RFC 0001), so a group lands in three pull requests deployed in the order under Adding a group. The checklist alone is what failed Bark, so it is backed by a test: each consumer keeps a recorded tool list from the servers it federates, and CI fails when a tool in it belongs to no known group or carries no classification. The test lives with the consumer, so it catches the omission in the repository that forgot rather than the one that shipped.

### Classification

Every tool declares metadata beyond its schema. This is the main addition to Bark's contract and it is what lets the Agent and the Surface treat a search differently from a send.

| Field | Values | Used by |
|---|---|---|
| `kind` | `read`, `write` | Agent: which server's rules apply |
| `permission` | A permission id from RFC 0003 | Agent: whether to bind the tool at all |
| `capability_class` | `reservation`, `broadcast`, `change_request`, `record`, or absent for reads | Surface: which approval permission and policy apply |
| `reversible` | `true`, `false` | Surface: whether the class requires four eyes; UI emphasis |
| `idempotent` | `key`, `natural`, `no` | Agent and provider: whether an idempotency key is required |
| `effect` | One sentence, written for a human | Surface: the heading of the approval preview |

Rules:

- A tool on `openbark-actions` must declare `kind: write`, a `capability_class`, and a `permission`. The server refuses to start otherwise.
- A tool on the knowledge server must declare `kind: read`. A read tool may not change state, including upstream state that looks incidental such as marking something seen.
- `idempotent: no` is not allowed with `reversible: false`. An irreversible action that cannot be made idempotent cannot be safely retried, and the Agent does retry on resume.
- `effect` is read by a human at the moment they decide. It is not a description for the model, and it is reviewed as UI copy.

The Agent binds a write tool only when the member's permissions include its `permission` (RFC 0002), so a member who cannot act never sees the tool.

### Authentication

Bark's MCP server is public because it serves only public data. Both OpenBark servers require two things on every call:

- **A service credential.** A bearer token proving the caller is OpenBark's Agent. Each server has its own, and neither is known to the browser or the Surface's frontend.
- **A forwarded user assertion.** The signed assertion the Surface minted for the turn (RFC 0003). The server verifies the signature, audience, and expiry itself rather than trusting the Agent's verification, because the authorization decision belongs where the data is.

A read tool applies the assertion's permissions to what it returns; `kb` passes the role ids to the index as a filter (RFC 0005). A write tool checks the permission again before executing and rejects a call with no recorded approval reference (RFC 0006).

### Adding a group

1. **Server:** add the sub-server under `services/<id>/`, mount it, declare classification on every tool, add tests under `tests/services/<id>/`, and document the tools in the README.
2. **Agent:** add the label and hint pattern, with a unit test showing a representative query selects the group and an unrelated one does not.
3. **Surface:** add the id and toggle, and the permissions the group needs to the role map.
4. **slai-rfcs:** add the group to the table above.
5. For a write group, the adversarial evaluations in RFC 0002 must pass before it is enabled in production.

Deploy in the order above, so the callee is deployed first (RFC 0001). Between steps 1 and 2 a new read group is briefly ungrouped, which is harmless; a new write group is not bound at all until step 2, which is the safe direction.

Adding a tool to an existing group changes only the server, though the hint pattern should be reviewed if the tool covers new vocabulary, and a new write tool needs its own adversarial evaluation.

### Renaming or removing a group

As in Bark: add the new id alongside the old, move consumers over, then remove the old id in reverse order (Surface, Agent, server). The Agent and the Surface both drop unknown ids from the toggle list, so a stale client cannot break a request, but a member who had switched off the old id would have it silently turned back on, so a rename maps the old id to the new one for the length of the transition.

A write group's permission id is renamed the same way, and the role map keeps both until every role has been moved, because a permission that resolves to nothing fails closed and makes the tool invisible rather than throwing.

### Tool design guidelines

Bark's guidelines hold, with three changes.

- **Descriptions are written for the model.** The first line says what the tool answers or does, and an `Args:` block documents each parameter with an example. The Agent keeps the summary and `Args:` and drops `Returns:` prose.
- **Compact outputs.** Return what answers the question, not the whole upstream payload. Every token of tool output goes back to the model.
- **Failures are messages, not exceptions.** When an upstream is down or a lookup finds nothing, return a short plain-language status the model can relay. The Agent's verification depends on these being accurate, in both directions: a tool that reports success it did not achieve will produce an answer claiming an action was taken.
- **Read tools return provenance.** Every result carries a citation id, a source reference, a version or commit, and a last-modified timestamp, so the Agent can cite it and the guards can verify it resolves (RFCs 0002, 0005). A read tool that returns prose with no provenance cannot be cited and should not exist.
- **Write tools preview before they act.** A write tool exposes `preview` and `execute` as separate operations over the same arguments, and `preview` must not change anything. RFC 0006 defines the shapes.
- **Arguments are member data.** Read tool arguments are not retained beyond request metadata, as in Bark. Write tool arguments are audited, which splits bark RFC 0004's blanket rule rather than dropping it; RFC 0006 states the split and the reasoning.

### Deployment

Kennel runs both servers from the repository's flake and checks `GET /api/health` after each deploy. Neither is exposed outside the deployment's network; only the Agent reaches them.

### Development environment

Each server's own RFC covers its devenv: the read tools come up with the index (RFC 0005), and the write server runs on 5052 with every provider defaulting to a dry-run implementation in non-production profiles (RFC 0006). To develop against Bark's campus tools, configure `mcp-server` at `http://127.0.0.1:5050/mcp` as a foreign read server, or leave it pointed at production, which needs no credential because it is public.

## Alternatives Considered

- **One server for all tools, as Bark has.** Fewer deployments and one configuration. Rejected: read and write differ in authentication consequence, argument retention, and whether approval gates execution, and a single process means a mount mistake crosses that boundary silently. Bark's argument against splitting is about groups, not posture.
- **Keeping the server public and unauthenticated.** It would let other clients reuse the `kb` tools with no credential handling, as Bark's campus tools are reused. Rejected because the index holds private material: the authorization decision has to happen where the data is. Bark's public server remains available for public campus data.
- **Discovering groups and classification from the server at runtime.** FastMCP supports tags and metadata, so the Agent and Surface could read the group list instead of hard-coding it, which would remove the drift bark RFC 0004 documents. Bark defers this because hint patterns are hand-written and the Surface would need a route. Here classification is a stronger reason to do it eventually and a stronger reason not to do it yet: a permission requirement the Agent learns from the server at runtime is a permission requirement the server can change without review. Deferred, and if it lands, `permission` stays declared on the consumer side.
- **Letting the Agent be the only authorization check.** The Agent already verifies the assertion and binds by permission, so the servers could trust it. Rejected: a bug or a prompt injection that gets the Agent to call a tool it should not have bound is exactly the case the second check exists for, and it is cheap.
- **Allowing ungrouped tools as always-on.** No process at all. Rejected for Bark's reason, that it silently costs context and removes member control, and for a new one: an ungrouped write tool has no permission and no toggle.

## Open Questions

- Should `gov` be its own group or part of `kb`? Governance files are structured TOML rather than prose and answer a different kind of question, which argues for a group; they are also indexed material, which argues against.
- Is `forge` a read group over the Forgejo API, or should repository content be reached only through the index? Proposed: both, with the index for search and `forge` for live state such as open pull requests, which goes stale too quickly to index.
- Does the write server need per-tool rate limits in addition to the per-class limits in RFC 0006?

## Implementation Phases

**Read server**

- the knowledge server with the `kb` group over the index (RFC 0005), classification declared and enforced at startup, service credential and assertion verification
- Agent group labels and hint patterns, Surface toggles, group table rows
- `forge` and `gov`

**Write server**

- `openbark-actions` skeleton: classification enforcement, approval reference required, dry-run providers in the dev profile
- First group with its provider (RFC 0006), gated on the adversarial evaluations in RFC 0002
