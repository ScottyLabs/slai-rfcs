# RFC 0005: Knowledge and Retrieval

- **Status:** Draft
- **Author(s):** @qianxuege
- **Created:** 2026-10-03
- **Updated:** 2026-10-03
- **Affects:** openbark-knowledge, openbark-agent

## Overview

This RFC defines `openbark-knowledge`, the service that ingests ScottyLabs' institutional material, answers retrieval queries over it, and publishes the read MCP tools that reach it. It defines the source connector interface that makes a corpus pluggable, how documents are chunked, embedded, and refreshed, how access is filtered by role, what a retrieval result carries so an answer can cite it, and how retrieval quality is measured. Bark has no counterpart to this RFC.

## Motivation

Most of what OpenBark is for depends on retrieval. Answering how a service is deployed, what a decision was based on, which repositories a change affects, what an onboarding step requires, or what last year's event announcement said are all the same operation over different corpora. Bark's only vector search is over personal memory facts, so there is nothing to extend and the component is new.

Three properties have to be designed in rather than added later.

**Pluggable sources.** The corpora the team will want are not knowable now. RFCs, repository READMEs, governance files, meeting notes, pull requests, and runbooks are the obvious first ones, and the list will keep growing. A new corpus must be a connector, not a change to the index.

**Role-filtered access.** Not everything ScottyLabs writes down is for everyone. Filtering has to happen inside the query, before results reach a prompt, because a result the Agent has seen has already been disclosed.

**Citations that resolve.** An uncited answer about an RFC is worse than no answer, because it is confidently wrong about the thing the team relies on. Every result carries provenance, and the Agent verifies that what the model cited is what the index returned (RFC 0002).

## Goals

- Define the source connector interface and the process for adding a corpus
- Define ingestion: chunking, embedding, metadata, incremental refresh, and deletion
- Define the retrieval query contract, including role filtering and hybrid search
- Define what a result carries so the Agent can cite it with freshness
- Define how retrieval quality is measured and what regressions block a deploy

## Non-Goals

- The Agent's use of results and its citation verification (RFC 0002)
- MCP tool naming, classification, and federation, including how Bark's `mcp-server` is consumed alongside this one (RFC 0004)
- Writing to sources, which is an action (RFC 0006)
- Any particular corpus beyond the reference connector

## Detailed Design

### Stack

Python with FastAPI for the query API, FastMCP for the read tool groups, and a worker for ingestion, managed with `uv`. The query API and the MCP endpoint are one deployable: the tools are a thin facade over the same query path, with no separate state or credentials, which is why RFC 0001 does not split them into a fifth repository. PostgreSQL with pgvector holds documents, chunks, and embeddings, with a full-text index for the lexical half of hybrid search. Postgres is the only store: a separate vector database would be a second thing to operate for a corpus of this size, and the team already runs Postgres with pgvector for Bark.

### Source connector interface

A corpus is added by implementing one interface. The index knows nothing about where material comes from.

```python
class SourceConnector(Protocol):
    source_id: str              # stable, e.g. "git.cmu.dev/ScottyLabs/slai-rfcs"
    refresh_interval: timedelta # how stale a result may be before it is marked

    def list_documents(self, since: str | None) -> Iterable[DocumentRef]:
        """Documents changed since an opaque cursor. Full enumeration when None."""

    def fetch(self, ref: DocumentRef) -> Document:
        """Content, content type, and metadata."""

    def cursor(self) -> str:
        """Opaque position for the next incremental pass."""
```

A `Document` carries its text or binary content, a content type, a stable `document_id` within the source, a `version` (commit sha, revision, or content hash), a `last_modified`, a canonical URL a human can open, and `visibility_labels`.

**Visibility labels are the connector's responsibility.** A connector declares, per document, the labels that describe who may read it, for example `public`, `slai`, `leads`. It does not know about OpenBark's permissions; the index maps labels to the `kb.read.<corpus>` permissions in RFC 0003. A connector that cannot determine a document's visibility must label it with the most restrictive label the source allows, never the least.

The reference implementation is a Forgejo connector over git.cmu.dev, which covers RFCs, READMEs, documentation, and governance files, and takes its cursor from the repository's commit history. A connector is roughly a hundred lines; most of the work is visibility.

### Adding a corpus

1. Implement the connector under `src/openbark_knowledge/sources/<id>/`, or in its own repository as an installable package, with tests over recorded fixtures.
2. Register it with its label-to-permission mapping and its refresh interval.
3. Add the `kb.read.<corpus>` permissions to the Surface's role map (RFC 0003).
4. Add questions covering the corpus to the evaluation set, and record the baseline.
5. Backfill, then enable incremental refresh.

A corpus is not considered live until step 4. Without it, a regression in retrieval quality caused by the new material is invisible.

### Ingestion

- **Chunking** is structure-aware. Markdown splits on heading boundaries, keeping the heading path with the chunk, because a chunk from an RFC is far more useful when it knows it came from "RFC 0004 > Adding a group". Code and TOML split on top-level definitions. Prose with no structure falls back to a fixed window with overlap. Chunks target roughly 500 tokens.
- **Metadata** on every chunk: `source_id`, `document_id`, `version`, `last_modified`, `heading_path`, `content_type`, `visibility_labels`, and the canonical URL. Every one of these is a filter the query may use.
- **Embedding** is per chunk, with the model recorded alongside. Startup refuses to serve a table whose embedding shape or model does not match the configured one, as bark RFC 0002 does for memory. A model change is a re-index, not a migration.
- **Incremental refresh** runs per source on its interval, asking the connector for changes since the stored cursor. A changed document's chunks are replaced atomically, so a query never sees half of two versions.
- **Deletion propagates.** A document absent from a full enumeration, or reported deleted, has its chunks removed within one refresh cycle. A corpus whose connector cannot report deletions is re-enumerated fully on a slower schedule, because stale material that no longer exists is the failure mode most likely to produce a confidently wrong answer.
- **Webhooks where available.** git.cmu.dev can notify on push, which makes the RFC and documentation corpora near-real-time. The interval remains the fallback.

### Query contract

```
POST /query
{
  "text": "how does a new tool group get registered",
  "roles": ["member", "slai"],
  "filters": { "source_id": ["..."], "content_type": ["markdown"],
               "last_modified_after": "2026-01-01" },
  "limit": 8
}
```

- **Role filtering is applied inside the query**, as a predicate on `visibility_labels` resolved through the label-to-permission map. It is not a post-filter and it is not optional: a query with no roles returns only `public` material. The index never receives the member's subject, only role ids (RFC 0002).
- **Hybrid search** runs lexical and vector retrieval and fuses the rankings. Lexical matters more here than in a general corpus because institutional material is full of exact identifiers (`AGENT_SHARED_SECRET`, `TOOL_GROUP_LABELS`, `RFC 0004`) that embeddings blur.
- **Reranking** is a cross-encoder over the fused candidates, returning the top `limit`. It is the single biggest quality lever and the main latency cost, so it is configurable and can be disabled for a cheaper path.
- **Results** are chunks with their metadata, a score, and a `citation_id` stable for the turn. The Agent cites by `citation_id` and the guards verify that every cited id was returned (RFC 0002).
- **Freshness** is computed, not stored: a result whose `last_modified` is older than its source's `refresh_interval` by a configurable factor is flagged `stale`, and the Agent surfaces that in the answer.

`GET /sources` returns each source with its last successful refresh, cursor age, and document count, which is what the Surface's health view and the Agent's `/api/health` read.

### Relationships between documents

Answering "what would be affected if I add an MCP tool group" needs more than nearest-neighbour search: it needs to know that bark RFC 0004's group table names three repositories and that each has a file that must change. Rather than build a general knowledge graph, the index extracts a small set of typed links during ingestion:

| Link | Extracted from |
|---|---|
| Document cites document | RFC cross-references, relative links |
| Document names repository | Repository names and URLs in text |
| Document names identifier | Code identifiers, environment variables, and route paths in backticks |
| Document supersedes document | Explicit status lines, as in the RFC process |

Links are stored as rows and exposed as a second query (`POST /links`) that walks them from a starting document to a bounded depth. This covers the cross-repository impact question without committing to a graph store, and the extraction is a regex-and-parser pass that can be improved independently.

This is deliberately the least certain part of the design. It is scoped so that if link extraction turns out to be weak, retrieval still works and only the impact-analysis use case is diminished.

### Evaluation

Retrieval quality is measured, because it cannot be reviewed by reading a diff.

- **A labelled set** of questions with the documents that should be retrieved, maintained alongside the corpora and extended whenever a corpus is added or a retrieval bug is found.
- **Metrics:** recall@k and mean reciprocal rank here, and citation accuracy and groundedness in the Agent, which runs the answer-side half against the same labelled set (RFC 0002).
- **A fixture corpus** committed to the repository, so the suite is reproducible offline and in CI.
- **A regression gate:** a drop in recall@8 beyond a threshold on the fixture corpus fails CI. Changes to chunking, embedding, fusion, or reranking are expected to move it and must report the movement in the pull request.

### Configuration

| Variable | Purpose |
|---|---|
| `DATABASE_URL` | Documents, chunks, embeddings, links (required) |
| `EMBEDDING_MODEL`, `EMBEDDING_API_KEY` | Chunk embeddings (required) |
| `RERANK_MODEL`, `RERANK_API_KEY` | Reranking (optional; disabled without it) |
| `SOURCES` | Enabled connectors with their label maps and intervals (required) |
| `FORGEJO_TOKEN` | git.cmu.dev read access for the reference connector |
| `STALE_FACTOR` | Multiple of `refresh_interval` past which a result is stale. Default 2 |
| `PORT` | Query API and `/mcp`. Default 5053 |

Embedding and reranking send chunk text to a provider, so the provider allowlist and zero-retention requirement in RFC 0002 applies to them as much as to the chat model.

### Development environment

`devenv up` starts PostgreSQL with pgvector, the query API, and the MCP endpoint on 5053. A `seed` task ingests the committed fixture corpus, so the whole stack runs locally with no credentials and no network. Ingesting a real source needs `FORGEJO_TOKEN` from OpenBao.

## Alternatives Considered

- **A dedicated vector database.** Purpose-built indexes, better scaling, less to tune. Rejected for now: the corpus is thousands of documents, not millions, hybrid search wants the lexical index and the vectors in one query, and Postgres with pgvector is already operated by the team. The query contract hides the store, so this can change without touching the Agent.
- **Vector-only retrieval.** Simpler, one index, no fusion. Rejected because institutional material turns on exact identifiers, and a question naming `TOOL_GROUP_LABELS` must find the document containing that string even when the surrounding prose is unlike the question.
- **Retrieval at answer time with no index, searching sources live.** Always fresh, no ingestion, no deletion problem. Rejected: it cannot rerank across corpora, it is slow and rate-limited, and most sources have no useful search. Freshness is addressed with webhooks and staleness marking instead.
- **Filtering results after retrieval, in the Agent.** It would let the index stay simple and unaware of roles. Rejected: a result returned to the Agent has already been disclosed, and the Agent is where prompt injection pressure is highest. Filtering belongs in the query.
- **Letting connectors default to permissive visibility.** Faster to add a corpus, and most ScottyLabs material is internally open. Rejected: a connector that guesses wrong in the permissive direction leaks, and the failure is silent. Most restrictive by default means the failure is someone asking why they cannot see a document.
- **A general knowledge graph over all extracted entities.** It would answer impact questions much better and is the interesting version of the problem. Deferred to typed links because a graph needs entity resolution to be useful, and a half-working graph is harder to debug than a table of links. The link rows are a subset a graph could be built from later.
- **Promoting facts from personal memory into the index.** Covered in RFC 0002: memory has no provenance and no access control, and shared knowledge needs both.

## Open Questions

- Which corpora at what visibility? The first pass is `slai-rfcs`, repository READMEs and docs, and governance, all at organization-wide visibility. Meeting notes are the first corpus likely to need something narrower, and the first real test of the label mapping.
- Where do meeting notes live, and does that source support incremental enumeration and deletion?
- Is a hosted reranker acceptable for private chunks, or does reranking need a self-hosted model? This gates whether reranking ships enabled.
- How large does the labelled evaluation set need to be before the regression gate is meaningful rather than noisy? Fifty questions is a guess.

## Implementation Phases

**Index**

- Schema, structure-aware chunking, embedding with shape enforcement, the query API with hybrid search and role filtering
- Fixture corpus, labelled evaluation set, CI regression gate

**First corpus**

- Forgejo connector over git.cmu.dev with visibility labels, backfill, incremental refresh, and push webhooks
- `kb` tool group and the Agent's retrieval stage (RFCs 0004, 0002)

**Relationships**

- Typed link extraction and the `POST /links` query
- Reranking, if the provider question resolves

**Further corpora**

- Governance, meeting notes, pull requests, runbooks, each following the five-step process above
