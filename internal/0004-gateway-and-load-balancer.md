# RFC 0004: Gateway and Load Balancer DRAFT

- **Status:** Super Duper Draft
- **Author(s):** Mofan Yu, Benson Chou, Ivan Chen, Aaron Zhu
- **Created:** 2026-10-24
- **Updated:** 2026-10-24
- **Affects:** the API client part of other apps?

## Overview

The gateway has a range of responsibilities.

## Motivation

- Unnatural split of GPUs and services
- High demand is bad
- Future scaling

## Goals

- Define what apps must do to call the gateway
- Plan to call the worker optimal for benchmarking
- Handle large loads of requests
- Store both secrets and metrics

## Non-Goals

- Specifying which cards/models/workers are dedicated to each pool (RFC 0003)

## Detailed Design

FOR STARTERS WE WILL USE LITELLM!

### App access and responsibilities

- Build the request with context and call LiteLLM.
- Choose either `sl-chat` or `sl-code`.
- Provide a student ID for potential rate limiting.
- Respect a 429 error.

Sample request:

```http
POST /v1/chat/completions
Authorization: Bearer <this-apps-virtual-key>
x-litellm-end-user-id: student-123
Content-Type: application/json

{"model":"sl-chat","messages":[{"role":"user","content":"Hi"}]}
```

### Load balancer

The gateway should make the most out of 2 sl-chat endpoints and 3 sl-code endpoints (RFC 0003). The routing strategy used will probably be the same for both pools. The dev slice will not use routing.

- Start with LiteLLM’s built-in “least-busy” strategy. LiteLLM gathers its own metrics. However, it may result in wasting a worker’s cached prompt when it switches to a different endpoint. (so we lose session affinity)
- Set 4 max parallel requests for each sl-code endpoint and 16 for each (production) sl-chat endpoint (from RFC 0003).
- Consider using different built-in routing strategies for the benchmark. LiteLLM routing docs: [https://docs.litellm.ai/docs/routing](https://docs.litellm.ai/docs/routing). In the future, we may switch to a more informed routing.
- Also switch to another endpoint if one returns an error.

### 429

When all available paths are full, return HTTP 429, aka “busy, retry later”. Probably include a “Retry-After”, a suggestion for the app, based off a random and exponential delay up to `min(30s, 1s × 2^attempt)`.

### 3080 allocation

In certain situations, such as Bark’s release, light-app traffic might overwhelm the 2080 slices. Then temporarily move one 3080 to sl-chat by changing its sl-code assignment to sl-chat. Test for cleanness of this switch.

### Data collecting and storage

LiteLLM collects some metrics, but the gateway is still responsible for some other data collection.

| Type of Data | Method of collection | Where it’s stored |
| --- | --- | --- |
| App/student identity and permissions | Supplied by the request | PostgreSQL |
| Request details, time and token usage | LiteLLM gathers this | PostgreSQL, maybe Prometheus |
| Gateway metrics | `/metrics` and `/health` | Prometheus, Loki, Grafana idk |

## Alternatives Considered

- Portkey Gateway offers retries and fallbacks but weaker key and budget management. Bifrost was also raised for consideration...
- Cross-pool fallback preserves availability but may send requests to an unsuitable model.

## Open Questions

- Should the permissions be stored on a self-contained gateway machine or ScottyLabs’ shared Postgres?
- Which components are designed to be reworked after the first initial layout and how do we minimize the disruption

## Implementation Phases

I mean this section would probably belong in RFC 0001
