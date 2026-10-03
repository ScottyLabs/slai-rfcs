# OpenBark RFCs

RFCs for OpenBark go in this folder, numbered from `0001`. See the [repository README](../README.md) for the process and [template.md](../template.md) to start one.

OpenBark is an internal assistant for ScottyLabs members and a shared platform the team builds modules on top of. It spans four repositories. RFC 0001 describes how they fit together, each service has its own RFC, and RFCs 0005 and 0006 define the two extension points that have no Bark counterpart: knowledge sources and actions.

| Repository | Role |
|---|---|
| `openbark-surface` | Web app and BFF: sign-in, authorization, chats, approvals, audit |
| `openbark-agent` | Planning, retrieval, model execution, verification, memory, run state |
| `openbark-knowledge` | Source ingestion, retrieval, and the read MCP tools over them |
| `openbark-actions` | Approval-gated write MCP tools and their providers |

RFCs 0001, 0004, and 0006 are contract RFCs: they are agreements between services, so their Affects lists are long and they should change rarely. RFCs 0002, 0003, and 0005 are home RFCs, each designing one service. A tool server appears in both kinds, in the contract it implements and the domain it serves; RFC 0001 explains the convention.

| Number | Title | Kind | Affects | Status |
|--------|-------|------|---------|--------|
| 0001 | [System Architecture](./0001-system-architecture.md) | Contract | all | Draft |
| 0002 | [Agent](./0002-agent.md) | Home | openbark-agent | Draft |
| 0003 | [Surface](./0003-surface.md) | Home | openbark-surface | Draft |
| 0004 | [MCP Tools](./0004-mcp-tools.md) | Contract | openbark-knowledge, openbark-actions, openbark-agent, openbark-surface | Draft |
| 0005 | [Knowledge and Retrieval](./0005-knowledge-and-retrieval.md) | Home | openbark-knowledge, openbark-agent | Draft |
| 0006 | [Actions, Approvals, and Audit](./0006-actions-and-approvals.md) | Contract | openbark-actions, openbark-agent, openbark-surface | Draft |

## Relationship to Bark

OpenBark reuses Bark's shape, including its separate-repository model, and departs from it where the trust posture differs. Each departure is recorded in the Alternatives Considered section of the RFC that makes it, so a Bark reviewer can see why the decision flipped.

| Bark decision | OpenBark | Where |
|---|---|---|
| Identity is pseudonymous past the Surface (`oidc:<sha256(...)>`) | A short-lived signed assertion carrying subject and roles | 0001, 0003 |
| The MCP server is public, unauthenticated, and read-only | OpenBark's own servers are authenticated and split read from write | 0001, 0004 |
| Tool arguments are never logged | Read arguments are not retained; write arguments are audited | 0004, 0006 |
| Models are routed freely through OpenRouter | Providers are allowlisted and zero-retention | 0002 |
| A chat is private or public to the internet (`is_public`) | Visibility is scoped, never public, and citations are re-authorized per viewer | 0003 |

## Reuse of Bark's MCP server

OpenBark does not re-wrap CMU campus data. Bark's `mcp-server` is configured as a third, foreign tool server alongside OpenBark's two, which is bark RFC 0001's claim that the MCP server "is useful beyond Bark" coming true. RFC 0001 defines the four federation rules this needs, and RFC 0004 implements them: group ids are unique across servers, a foreign server's classification comes from configuration, the member's assertion is never forwarded to it, and its outage degrades rather than fails. Bark is unchanged.
