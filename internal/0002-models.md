# RFC 0002: Models

- **Status:** Draft
- **Author(s):** @jiyas
- **Created:** 2026-10-23
- **Updated:** 2026-10-23
- **Affects:** `cmugpt-agent`, and future AI Team apps that call the platform

## Overview

The coding agent uses Qwen3.8-27B, served under the name `sl-code`, at low to medium thinking effort. If it proves too slow, `sl-code` switches to Qwen3-Coder-30B-A3B. Bark and the other light apps use whichever ~9B model is fastest while passing Bark's eval (Qwen3.5-9B, Granite 4.1 8B or Ministral 3 8B), served under the name `sl-chat` with thinking always off. Apps only ever see the two names (RFC 0001), so either model can be swapped later without changing any app.

## Motivation

The platform's GPUs are 3× RTX 3080 (20 GB each) and 2× RTX 2080 Ti (22 GB each). The two kinds of app need very different things:

- **Bark** is a light helper. It answers campus questions with 1–2 tool calls and short replies. It runs on GPT-5.6 Luna today. It needs reliable tool calls and fast answers, and no heavy reasoning.
- **The coding agent** (up to 200 users) sends long contexts (files, diffs) and makes 5–30 model calls per task. It needs strong coding quality at a reasonable speed.

One model can't serve both well on our cards. A model strong enough for coding is too slow and requires too much memory for many short Bark requests, and a model fast enough for Bark is too weak for coding.

## Goals

- **Bark (`sl-chat`):** at least 95% tool-call agreement with GPT-5.6 Luna and 98% valid output on real Bark questions (method in RFC 0005), and first token in under 2 seconds.
- **Coding agent (`sl-code`):** first token in under 3 seconds and at least 30 tokens per second per user at normal load. The task pass rate target is set once the task set exists (RFC 0005).
- Each model fits its assigned cards with room for several requests at once (RFC 0003).

## Non-Goals

- GPU machines, drivers, passthrough and vGPU - managed by OCC provides these.
- The inference engine and GPU layout (RFC 0003) and routing (RFC 0004).
- Embedding, reranker and moderation models. This depends on RFC 0001's Bark's OpenAI calls.
- Models for the other 2–3 apps until their needs are described. They default to `sl-chat`.

## Detailed Design

### Models

| Name | Model | Build | GPU memory for weights | Context cap | Thinking |
| --- | --- | --- | --- | --- | --- |
| `sl-code` | Qwen3.8-27B (Apache 2.0) | 4-bit build that vLLM runs on Ampere (AWQ/GPTQ) | ~15–16 GB of 20 GB | 32k tokens per request | Low to medium effort; higher with "think harder" |
| `sl-chat` | Fastest that passes Bark's eval: Qwen3.5-9B, Granite 4.1 8B or Ministral 3 8B | 4-bit AWQ | ~5–6 GB | 8k tokens per request | Always off |

- **Pinning:** each model is pinned to an exact Hugging Face revision in the worker list (RFC 0003), so it never changes without us knowing - vLLM downloads the newest files every time a worker starts, so without pinning, it will automatically pull revisions from the model repository. Pinning allows us to control the exact version we run. This should be updated regularly to keep up model updates
- **Qwen3.8-27B:**
  - The newest Qwen generation, with strong reported coding-agent results.
  - Its 4-bit weights fit one 20 GB card.
  - Three of every four layers use linear attention, which should make long contexts much cheaper in GPU memory than a normal 27B model. We haven't measured that yet, and it decides how many coding requests fit per card.
- **The ~9B candidates:**
  - Bark needs speed, not reasoning, so all three go through Bark's eval (RFC 0005), and the fastest one that passes becomes `sl-chat`.
  - At ~5–6 GB, any of them leaves room for dozens of short Bark requests per card.

### Thinking

The engine separates reasoning from the answer (vLLM's `--reasoning-parser`), so reasoning text never breaks an app's output parsing.

- **`sl-chat`:** thinking is always off. Apps send the model's thinking switch set to off on every request.
- **`sl-code`:** low to medium reasoning effort by default. Lower it if answers are too slow against the Goals.
  - The coding agent offers a "think harder" toggle. When a student turns it on, that one request is sent at a higher effort.
- **Both:** apps strip earlier turns' reasoning from the conversation history before resending it.

### What apps change

Apps ask for `sl-chat` or `sl-code` instead of a provider model ID, and send the thinking settings above. The full list of what apps must do is in RFC 0004.

## Alternatives Considered

- **One model for every app.** It would be simplest and pool every card, but coding needs a large model and Bark needs a fast one. Neither app is served well by a compromise.
- **Qwen3-Coder-30B-A3B for coding.** It's several times faster per token, because only ~3B parameters are active at a time. But it's an older generation with lower coding quality, and at ~17–22 GB it needs two cards to leave room for context. It's the fallback if Qwen3.8-27B misses the speed goal.
- **Devstral Small 24B or Mistral Small 4 for coding.** Both are built for agentic coding and fit one card. The team prefers Qwen, and long contexts cost these models the normal amount of memory.
- **Gemma 4 31B for coding.** It has strong coding results in reviews, but at ~18–19 GB at 4-bit it's too large for a 20 GB card once requests need memory.
- **A 12–14B model for Bark** (Gemma 4 12B, Qwen3-14B, Ministral 3 14B). It reasons better, but it's slower and fits about half the simultaneous requests of a 9B. It's the step up only if no 9B passes Bark's eval.
- **DeepSeek for Bark.** Its models are strong and MIT-licensed, but the current open DeepSeek V4 Flash needs roughly 130–175 GB of GPU memory. The small distilled versions fit but are thinking-heavy, which Bark doesn't need.

## Open Questions

- Is there a 4-bit build of Qwen3.8-27B that the vLLM version we pin runs on a 3080? It's very new, and some published builds (e.g. NVFP4) target newer GPUs.
- Does Qwen3.8-27B reach 30 tokens/s per user on a real 3080? If not, switch `sl-code` to Qwen3-Coder-30B-A3B.
- How much GPU memory does a 32k-token request use with Qwen3.8-27B's design?
- Which of the three ~9B candidates is fastest while still passing Bark's eval with thinking off?
- What do the other 2–3 apps need?

