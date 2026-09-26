# RFC 0004: MCP Server

- **Status:** Draft
- **Author(s):** @krishsax
- **Created:** 2026-09-26
- **Updated:** 2026-09-26
- **Affects:** mcp-server, cmugpt-agent, cmugpt-surface

## Overview

This RFC defines the design of `mcp-server`, which publishes CMU campus data as Model Context Protocol tools. It also defines the tool contract that the Agent and the Surface depend on: how tools are named, what a tool group is, and which places must change together when a group is added. It resolves the current gap where the `bus` group is published but unknown to the Agent and the Surface.

## Motivation

A tool group is an id that three repositories share, and nothing enforces that they agree:

- **mcp-server** creates a group by mounting a sub-server with a prefix (`main_mcp.mount(bus_mcp, prefix="bus")`), which names every tool `bus_<action>`.
- **cmugpt-agent** lists known groups in `TOOL_GROUP_LABELS` and matches queries to them with `_GROUP_HINT_RES`, both in `src/cmugpt/mcp_tools.py`.
- **cmugpt-surface** lists the groups a user can switch off in `AGENT_TOOL_IDS` (`apps/server/src/lib/agentTools.ts`).

When the bus sign tools shipped, only the first list changed. The Agent treats tools with an unknown prefix as ungrouped and always binds them. So all five `bus_*` schemas are sent to the model on every turn, including greetings, and users can't switch the bus tools off. Nothing failed, which is why the gap went unnoticed.

## Goals

- Document the MCP server's structure and data sources
- Define tool naming, the tool groups, and the change process across repositories
- Set guidelines for tool design that keep the model's context small and answers grounded
- Resolve the `bus` gap and bring the repository in line with RFC 0001's development environment

## Non-Goals

- How the Agent chooses among tools (RFC 0002)
- Authentication for the MCP server, which stays public while it serves only public data (RFC 0001)
- Data sources outside MCP

## Detailed Design

### Stack and structure

Python with FastMCP, `httpx`, `aiohttp`, and `pydantic`, managed with `uv` and packaged for Kennel with Nix. `core/app.py` defines the root server, `main_mcp`. Each data source is a FastMCP sub-server under `services/<group>/`, and `main.py` mounts each one under its group prefix, adds `GET /api/health`, and serves streamable HTTP on `PORT` (default 5050) at `/mcp`.

The server is stateless and read-only. It holds no user data, and its only configuration is optional: `GITHUB_TOKEN` (raises the GitHub API rate limit for `guide`) and `BUS_SIGN_API_URL` (overrides the bus sign API).

### Tool groups

| Group id | Label | Data source | Tools |
|---|---|---|---|
| `eats` | CMUEats | `api.cmueats.com` | 6 |
| `maps` | CMUMaps | `rust.api.maps.scottylabs.org` | 4 |
| `courses` | CMUCourses | `course-tools.apis.scottylabs.org` | 8 |
| `guide` | CMU Guide | `ScottyLabs/cmu-guide` on GitHub | 5 |
| `bus` | CMU Bus Sign | `bus-sign.scottylabs.org` | 5 |

This table is the source of truth for group ids. Every row must appear in all three repositories:

| Repository | Where | What |
|---|---|---|
| mcp-server | `src/mcp_server/main.py` | `main_mcp.mount(<sub>, prefix="<id>")` |
| cmugpt-agent | `src/cmugpt/mcp_tools.py` | An entry in `TOOL_GROUP_LABELS` and a pattern in `_GROUP_HINT_RES` |
| cmugpt-surface | `apps/server/src/lib/agentTools.ts` and the chat UI | An entry in `AGENT_TOOL_IDS` and a toggle |

### Naming

- Every tool is named `<group>_<action>` in snake_case, for example `eats_get_location_hours` or `maps_get_path`. The prefix comes from the mount, so a sub-server names its tools just `<action>`.
- A group id is a short, lowercase word matching `[a-z]+`. It must not contain `_`, because the Agent finds a tool's group by matching the `<group>_` prefix.
- Every tool belongs to exactly one group. Ungrouped tools are not allowed, because they can't be switched off or narrowed and they cost context on every turn.

### Adding a group

1. **mcp-server:** add the sub-server under `services/<id>/`, mount it, add tests under `tests/services/<id>/`, and document the tools in the README. Deploy.
2. **cmugpt-agent:** add the label and a hint pattern, with a unit test showing that a representative query selects the group and an unrelated one doesn't. Deploy.
3. **cmugpt-surface:** add the id and the toggle. Deploy.
4. **slai-rfcs:** add the group to the table above.

The order follows RFC 0001's "deploy the callee first" rule. Between steps 1 and 2 the new tools are briefly ungrouped, which is harmless.

Adding a tool to an existing group only changes the MCP server, though the Agent's hint pattern should be reviewed if the tool covers new vocabulary.

### Renaming or removing a group

Add the new id alongside the old one, move consumers over, then remove the old id in reverse order: Surface, then Agent, then MCP server. The Agent and the Surface both drop unknown ids from `disabled_tools`, so a stale client can't break a request. But a user who had switched off the old id would have that tool silently turned back on, so a rename must map the old id to the new one while the transition lasts.

### Tool design guidelines

- **Read-only, public data.** Tools don't change state or return data about specific users.
- **Descriptions are written for the model.** The first line says what the tool answers, and an `Args:` block documents each parameter's format with an example. The Agent keeps the summary and `Args:` and drops `Returns:` prose, so anything the model needs goes in those two places.
- **Compact outputs.** Return what answers the question, not the whole upstream payload, because every token of tool output goes back to the model.
- **Failures are messages, not exceptions.** When the upstream is down or a lookup finds nothing, return a short plain-language status the model can relay. The `bus` tools do this with `status` values such as `empty` and `input_error`. The Agent's check for false "lookup failed" claims depends on these being accurate.
- **Arguments are user data.** They can quote user messages, so they aren't logged beyond request metadata.

### Deployment

Kennel runs the `api` service at `api.mcp-server.scottylabs.org` and checks `GET /api/health` after each deploy.

### Development environment

`devenv up` runs the server on port 5050. Without Nix, run `uv sync` and then `uv run mcp-server`. The server has no required secrets, so it runs without OpenBao access. To test it with a local Agent, set the Agent's `MCP_SERVER_URL` to `http://127.0.0.1:5050/mcp` (RFC 0001).

## Alternatives Considered

- **Declaring groups once, in the MCP server.** FastMCP supports tags and server metadata, so the Agent and the Surface could discover groups at runtime instead of hard-coding them. That would remove the drift entirely. It is deferred because the Agent's hint patterns are written by hand for each group, and the Surface would need a route to fetch the list. The group table and the checklist cover the gap until then.
- **Allowing ungrouped tools as always-on.** It needs no process at all. It is rejected because it silently costs context on every turn and removes user control, as the `bus` case shows.
- **One MCP server per group.** It gives stronger isolation. But it multiplies deployments and Agent configuration for small, similar services.

## Implementation Phases

**Close the `bus` gap**

- cmugpt-agent: register `bus` with its own hint pattern (RFC 0002)
- cmugpt-surface: add `bus` to `AGENT_TOOL_IDS` and a toggle (RFC 0003)

**Repository alignment**

- Point the devenv readiness check at `/api/health` (it currently probes `/health`)
- Add `secretspec.toml` with optional `GITHUB_TOKEN` and `BUS_SIGN_API_URL`
- Update the README: port 5050, `uv` only (it still describes Poetry, Docker, and ports 8000 and 8001), and the five groups
