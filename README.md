# SLAI RFCs

Request for Comments (RFC) documents for ScottyLabs AI (SLAI). RFCs propose and record significant technical decisions across the team's projects, and they stay here as a record of why things are the way they are.

Repository: <https://git.cmu.dev/ScottyLabs/slai-rfcs>

## Repository structure

RFCs are grouped by project. Each project has its own folder and its own numbering, starting at 0001.

```
slai-rfcs/
├── README.md                      # this file: structure, process, RFC index
├── template.md                    # copy this to start an RFC
├── bark/                          # Bark, the CMU campus assistant
│   ├── 0001-system-architecture.md
│   ├── 0002-agent.md              # cmugpt-agent
│   ├── 0003-surface.md            # cmugpt-surface
│   └── 0004-mcp-server.md         # mcp-server
├── integration/                   # Integration (no RFCs yet)
├── internal/                      # Internal (no RFCs yet)
├── openbark/                      # OpenBark, the internal assistant and platform
│   ├── 0001-system-architecture.md
│   ├── 0002-agent.md
│   ├── 0003-surface.md
│   ├── 0004-mcp-tools.md
│   ├── 0005-knowledge-and-retrieval.md
│   └── 0006-actions-and-approvals.md
├── slc/                           # SLC (no RFCs yet)
└── slrp/                          # SLRP (no RFCs yet)
```

## Projects

### Bark

Bark spans three repositories. RFC 0001 describes how they fit together, and each service has its own RFC.

| Repository | Role |
|---|---|
| [cmugpt-surface](https://git.cmu.dev/ScottyLabs/cmugpt-surface) | Web app and backend-for-frontend API: sign-in, chat history, preferences |
| [cmugpt-agent](https://git.cmu.dev/ScottyLabs/cmugpt-agent) | LLM agent: tool selection, answer checks, per-user memory |
| [mcp-server](https://git.cmu.dev/ScottyLabs/mcp-server) | MCP server publishing CMU campus data as tools |

| Number | Title | Affects | Status |
|--------|-------|---------|--------|
| 0001 | [System Architecture](./bark/0001-system-architecture.md) | all | Draft |
| 0002 | [Agent](./bark/0002-agent.md) | cmugpt-agent | Draft |
| 0003 | [Surface](./bark/0003-surface.md) | cmugpt-surface | Draft |
| 0004 | [MCP Server](./bark/0004-mcp-server.md) | mcp-server, cmugpt-agent, cmugpt-surface | Draft |

### Integration

No RFCs yet.

### Internal

No RFCs yet.

### OpenBark

OpenBark is an internal assistant for ScottyLabs members and a shared platform the team builds modules on top of. It reuses Bark's shape and inverts its trust posture: members are authorized rather than only authenticated, the material it answers from is private and cited, and it takes approved actions on a member's behalf. RFC 0001 describes the architecture and lists the extension points.

OpenBark spans four repositories, and also consumes Bark's `mcp-server` as a tool server rather than re-wrapping campus data.

| Repository | Role |
|---|---|
| [openbark-surface](https://git.cmu.dev/ScottyLabs/openbark-surface) | Web app and BFF: sign-in, authorization, chats, approvals, audit |
| [openbark-agent](https://git.cmu.dev/ScottyLabs/openbark-agent) | Planning, retrieval, model execution, verification, memory, run state |
| [openbark-knowledge](https://git.cmu.dev/ScottyLabs/openbark-knowledge) | Source ingestion, retrieval, and the read MCP tools over them |
| [openbark-actions](https://git.cmu.dev/ScottyLabs/openbark-actions) | Approval-gated write MCP tools and their providers |

| Number | Title | Affects | Status |
|--------|-------|---------|--------|
| 0001 | [System Architecture](./openbark/0001-system-architecture.md) | all | Draft |
| 0002 | [Agent](./openbark/0002-agent.md) | openbark-agent | Draft |
| 0003 | [Surface](./openbark/0003-surface.md) | openbark-surface | Draft |
| 0004 | [MCP Tools](./openbark/0004-mcp-tools.md) | openbark-knowledge, openbark-actions, openbark-agent, openbark-surface | Draft |
| 0005 | [Knowledge and Retrieval](./openbark/0005-knowledge-and-retrieval.md) | openbark-knowledge, openbark-agent | Draft |
| 0006 | [Actions, Approvals, and Audit](./openbark/0006-actions-and-approvals.md) | openbark-actions, openbark-agent, openbark-surface | Draft |

Where OpenBark reverses a decision the Bark RFCs argue for, the reversal is recorded in that RFC's Alternatives Considered section. The [OpenBark README](./openbark/README.md) lists them.

### SLC

No RFCs yet.

### SLRP

No RFCs yet.

## When to write an RFC

Write an RFC when you want to propose:

- A change to a contract between services, such as an HTTP API, the MCP tool interface, or a shared identifier like a tool group id
- A new service, tool group, or external data source
- A change to how user data is stored, retained, or sent to third parties (models, embeddings, moderation)
- A change to authentication, secrets, or deployment
- A change to development processes or tooling that affects more than one repository

Bug fixes, documentation improvements, prompt tweaks, and changes contained in one service that don't alter a contract don't need an RFC. Open an issue or a pull request in the relevant repository instead.

## Process

1. **Draft:** copy [template.md](./template.md) into the project's folder as `####-short-title.md`, using the next free number in that folder, add it to the project's table under Projects, and open a pull request.
1. **Review:** the team discusses in PR comments, and the author revises. An RFC that affects a repository should be reviewed by at least one person who works on it.
1. **Accepted or Rejected:** an accepted RFC is merged with its status set to Accepted. A rejected RFC is closed, or merged with status Rejected if the reasoning is worth keeping.

When implementation details change, update the RFC and its **Updated** date. When a later RFC replaces an earlier one, set the earlier one's status to `Superseded by RFC ####`.

## Naming

Filenames are the number (zero-padded to 4 digits) and a kebab-case title, for example `bark/0005-conversation-export.md`. Branches use the `rfc/` prefix followed by the project and the filename, for example `rfc/bark/0005-conversation-export`. Within a project, refer to other RFCs by number ("RFC 0002"). Across projects, include the folder ("bark RFC 0002").

## Questions

Ask in the SLAI Slack channel (listed in [governance](https://git.cmu.dev/ScottyLabs/governance/src/branch/main/data/teams/slai.toml)) or open an issue in this repository.
