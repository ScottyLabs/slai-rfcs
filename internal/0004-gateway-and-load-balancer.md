# RFC 0004: Gateway and Load Balancer

- **Status:** Draft
- **Author(s):** Mofan Yu, Benson Chou
- **Created:** 2026-10-24
- **Updated:** 2026-10-30
- **Affects:** Clients of apps such as `cmu-gpt` and `cmu-agent`

## Overview

This RFC proposes a single LiteLLM gateway for apps using the self-hosted `sl-chat` and `sl-code` models. It defines how apps access the models, how requests are routed across workers, and how overload and failures are handled.

## Motivation

- **Split GPUs and services:** The two model pools run on different GPU types, so apps need one stable API.
- **High demand:** Concurrent requests can fill a pool; the gateway needs to distribute work and return a clear response when capacity is exhausted.
- **Future scaling:** Adding or replacing workers should require gateway configuration changes without changing each app's API integration.

## Goals

- Define app responsibilities.
- Suggest routing strategies.
- State how to handle failiure scenarios.
- Specify collection and storage of secrets and metrics.

## Non-Goals

- Specifying which cards/models/workers are dedicated to each pool (RFC 0003)
- Considering tests and benchmarking for optimal routing (RFC 0005).

## Detailed Design

LiteLLM will be the initial gateway in front of the vLLM workers. It will authenticate apps, apply configured limits, route requests within each model pool, and record usage. This gives us a working baseline for evaluating routing behavior, throughput, failure handling, and operational effort before revisiting the gateway choice.

### Starting configuration

Run LiteLLM with a `config.yaml`, PostgreSQL connection, and admin master key. Supply the database URL and master key through environment variables or the configured secrets mechanism. Register each vLLM server as a separate deployment under the public `sl-chat` or `sl-code` model name; its `api_base` is the worker URL and `model` is `hosted_vllm/<name-served-by-vllm>`. The dev slice is not in either production group. [LiteLLM vLLM setup](https://docs.litellm.ai/docs/providers/vllm), [virtual keys](https://docs.litellm.ai/docs/proxy/virtual_keys)

Enable LiteLLM's Prometheus callback so Prometheus can scrape gateway `/metrics`; also scrape worker metrics. Send gateway and worker logs to Loki with a log collector such as Grafana Alloy. Connect Prometheus and Loki as Grafana data sources for dashboards and investigation. These monitoring services do not make routing decisions. [LiteLLM metrics](https://docs.litellm.ai/docs/proxy/prometheus), [Loki log collection](https://grafana.com/docs/loki/latest/send-data/alloy/)


### App responsibilities

- **App virtual key:** Keep the app's LiteLLM key on its backend and use it when calling the gateway. Choose `sl-chat` or `sl-code` in each request according to the app's task.
- **HTTP 429s and failures:** Distinguish a rate limit from a temporarily full pool and use bounded retries when appropriate. Show a service-unavailable message when the gateway or model pool is down.
- **Student identity and conversation history:** Authenticate the student and send a stable student ID with each request. Store conversation history in the app and include the context needed for each model call.
- **Rate limiting**: Enforce per-student limits. In the future, apps may be responsible for managing a per-app request cap.

**Sample request**
```http
POST /v1/chat/completions
Authorization: Bearer <this-apps-virtual-key>
x-litellm-end-user-id: student-id
Content-Type: application/json

{"model":"sl-chat","messages":[{"role":"user","content":"Hi"}]}
```

### Load balancer

The gateway will distribute requests across two `sl-chat` endpoints and three `sl-code` endpoints (RFC 0003). The dev slice is exclusive to development testing.

- For both pools, start with LiteLLM’s built-in `least-busy` strategy, which uses its observed in-flight requests. However, requests from the same conversation may reach different endpoints, wasting a worker’s cached prompt.
- Limit each `sl-code` endpoint to 4 parallel requests and each production `sl-chat` endpoint to 16 (RFC 0003).
- Benchmark other [LiteLLM routing strategies](https://docs.litellm.ai/docs/routing) and revisit this choice if the results justify it.
- For retryable endpoint errors, try another available endpoint in the same pool.

### Failure scenarios

Apps may experience unavailable pools from outages, restarts, and errors. LiteLLM can route around a failed endpoint when another is available in the same pool. Optional background health checks let LiteLLM exclude unhealthy deployments before a user request reaches them.

| Failure | Effect | Response |
| --- | --- | --- |
| One model endpoint is down | Its pool has fewer available workers. | Retry on a healthy endpoint in the same pool, subject to LiteLLM's retry settings. |
| The `llm-vgpu` VM is down | Both production `sl-chat` endpoints are unavailable. | Report `sl-chat` as unavailable to the app. |
| The `llm-gpu` VM is down | All three `sl-code` endpoints are unavailable. | Report `sl-code` as unavailable to the app. |
| The gateway VM is down | No app can reach LiteLLM. | The app handles a connection error or an upstream proxy error; LiteLLM cannot return a response. |

An unavailable service is different from a rate-limiting error.

### Data collecting and storage

LiteLLM records usage and permissions; the monitoring stack collects operational metrics and logs.

| Type of Data | Method of collection | Where it’s stored |
| --- | --- | --- |
| App keys, permissions, and student IDs | Keys and permissions are configured in LiteLLM; apps send student IDs | PostgreSQL |
| (Secondary) Request metadata, time, and usage | LiteLLM records each call | PostgreSQL |
| Gateway and worker metrics | Prometheus scrapes `/metrics` endpoints | Prometheus; viewed in Grafana |
| Gateway and worker logs | A log collector forwards process logs | Loki; viewed in Grafana |

### 3080 allocation

If sustained light-app demand, such as Bark release, overwhelms the 2080 slices, we may temporarily assign one 3080 to `sl-chat`. This requires draining its `sl-code` worker, starting the correct chat model, and updating LiteLLM's configuration.

## Alternatives Considered

- **Other gateway software:** Portkey Gateway offers retries and fallbacks but weaker key and budget management. A custom gateway would provide full control but require us to build keys, limits, and streaming. nginx or Caddy with static keys would be simpler but would not provide per-student usage tracking or fallback.
- **Cross-pool fallback:** This helps preserves availability but may send requests to an unsuitable model or use more pool capacity than expected.
- **Extra VM for gateway:** A separate GPU VM for our gateway could relieve a worker but requires more management and does not prevent a gateway failiure from affecting every app.

## Open Questions

- Should the keys, IDs and permissions be stored on a self-contained gateway machine or ScottyLabs’ shared Postgres?
- Which components should be reworked after the initial layout and how do we minimize this process?