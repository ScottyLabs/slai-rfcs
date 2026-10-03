# RFC 0006: Actions, Approvals, and Audit

- **Status:** Draft
- **Author(s):** @qianxuege
- **Created:** 2026-10-03
- **Updated:** 2026-10-03
- **Affects:** openbark-actions, openbark-agent, openbark-surface

## Overview

This RFC defines how OpenBark changes things outside itself. It specifies the action provider interface, the four capability classes that providers implement, the preview and approval lifecycle, idempotency, blast-radius limits, and the audit record. Room booking, mass emailing, opening pull requests on git.cmu.dev, and reminders, task lists, and events are all providers against this contract rather than features in the Agent. Bark has no counterpart to this RFC; Bark's tools are read-only by design.

## Motivation

The features the team wants first are a room booking tool, a mass emailing tool, admin workflows, and git.cmu.dev integration. Built directly, each would need its own preview, its own authorization check, its own retry behaviour, and its own record of who approved it, and the second would copy the first badly. The parts that are hard about them are identical, and none of them are the part that looks hard.

What is actually hard is that a language model is choosing the arguments. The model reads retrieved documents, pull request bodies, and meeting notes written by other people and by automated systems, any of which can contain text addressed to it. Giving that model a tool that emails the organization is the dangerous configuration, and no amount of prompt engineering makes it safe. The containment is structural: nothing executes without a human approving a preview of exactly what will happen, and the arguments must trace to what the member asked for rather than to something the model read.

Specifying this once also means the fifth provider costs a schema and a test rather than a design.

## Goals

- Define the action provider interface, so a capability is a module
- Define the capability classes and what each guarantees
- Define the preview, approval, and execution lifecycle across the three components
- Define idempotency, so a resumed or retried run does not repeat an effect
- Define blast-radius limits and the kill switch
- Define the audit record
- Show that room booking and mass emailing are expressible, without specifying either

## Non-Goals

- Any concrete provider. Each gets its own RFC, which this one makes short
- The Agent's run suspension mechanics (RFC 0002) and the Surface's queue and audit views (RFC 0003), of which this defines the contract rather than the implementation
- Tool naming and classification metadata (RFC 0004)

## Detailed Design

### Action provider interface

```python
class ActionProvider(Protocol):
    action_id: str                  # "<group>_<action>", e.g. "rooms_reserve"
    capability_class: CapabilityClass
    permission: str                 # "action.preview.<class>" granted to callers
    reversible: bool
    Args: type[BaseModel]           # the typed argument schema

    def preview(self, args: Args, actor: Actor) -> Preview:
        """Describe what execute would do. Must not change anything."""

    def execute(self, args: Args, actor: Actor, idempotency_key: str) -> Result:
        """Perform the effect. Must be idempotent for a given key."""

    def limits(self) -> Limits:
        """Blast-radius bounds this provider enforces."""
```

A provider is registered on `openbark-actions`, which derives the MCP tool from `Args` and the classification metadata in RFC 0004 from the fields above. A provider may live in the service's own tree or in its own repository as an installable package, so adding a capability does not require merge access to the service that can email the organization (RFC 0001). The Agent and the Surface are unchanged when one is added.

**`preview` must not change anything.** This is the one invariant with no workaround. A provider whose upstream has no way to compute an effect without performing it (a reservation API with no availability check, a mailer with no recipient resolution) cannot implement `preview` honestly and must not be built against this contract until it can. Say so in the provider's RFC rather than approximating.

A `Preview` is what a human reads and approves:

```python
class Preview(BaseModel):
    summary: str                 # one line: "Email 214 members of ScottyLabs"
    details: list[tuple[str, str]]  # labelled fields shown in full
    scope: Scope                 # counted magnitude: recipients, rooms, repos
    warnings: list[str]          # irreversibility, unusual scope, stale inputs
    rendered: str | None         # exact outgoing content, where there is any
```

`scope` is a count with a unit, so the Surface can compare it against `limits()` and the approver can see magnitude without reading `details`. `rendered` carries the literal artifact where one exists, because approving a description of an email is not approving the email.

### Capability classes

A class is a family of effects that share a risk profile and therefore share policy. Four cover what the team has asked for and are intended to cover most of what comes next.

| Class | Effect | Reversible | Policy |
|---|---|---|---|
| `record` | Create or update an item OpenBark tracks: a task, a reminder, an event | Yes | Single approval; may be auto-approved by configuration |
| `change_request` | Propose a change to a versioned artifact: a pull request, a draft document | Yes, the proposal is not the change | Single approval; `rendered` is the diff |
| `reservation` | Hold a finite resource for a time window | Usually, by cancelling | Four eyes when the resource is shared; conflict check in `preview` |
| `broadcast` | Deliver a message to a resolved set of recipients | No | Four eyes always; recipient list enumerated in `preview`; rate limited; never auto-approved |

The mapping from the original feature list:

- **Room booking** is a `reservation` provider. `preview` reports the room, the window, and whether it conflicts; `scope` counts rooms.
- **Mass emailing** is a `broadcast` provider. `preview` enumerates the resolved recipients and renders the exact message; `scope` counts recipients.
- **git.cmu.dev integration** is a `change_request` provider. `preview` renders the diff and names the repository and branch.
- **Reminders, task lists, and events** are `record` providers, and are the right first implementation because they are reversible and internal.

Neither room booking nor mass emailing is specified here. What this RFC fixes is that when they arrive, their RFCs describe an upstream integration and a `preview`, and inherit approval, idempotency, limits, and audit.

A new class is added only when an effect does not fit an existing one, because a class is a policy and every class is another policy to review. Adding one requires an RFC amending this table.

### Lifecycle

```mermaid
sequenceDiagram
    participant M as Member
    participant S as Surface
    participant A as Agent
    participant W as openbark-actions
    participant P as Provider

    M->>S: message
    S->>A: stream request + assertion
    A->>W: preview(args)
    W->>P: preview(args, actor)
    P-->>A: Preview
    A->>S: action_preview, then action_pending(run_id)
    Note over A: run persisted, nothing changed
    S->>M: preview shown; queued for approvers
    M->>S: approve (an approver with the permission)
    S->>S: record approver, write audit
    S->>A: resume(run_id, approval_ref)
    A->>A: re-check args against preview
    A->>W: execute(args, idempotency_key, approval_ref)
    W->>P: execute(...)
    P-->>A: Result
    A->>S: action_result, audit record
    A->>S: answer continues
```

Invariants:

1. **No execution without an approval reference.** `openbark-actions` rejects an `execute` call whose `approval_ref` does not resolve to a recorded approval for that run, action, and argument hash. The Agent checks this too; the server checks it because it is the last line.
2. **The approved bytes are what execute.** The argument hash is computed at preview and re-checked at execution. If retrieval or the conversation changed in between, the action fails rather than drifting.
3. **The approver is a person with the permission.** `action.approve.<class>`, separate per class (RFC 0003). Classes configured for four eyes require an approver who is not the requester.
4. **Arguments must trace to the member's request.** The Agent blocks an action whose arguments appear only in retrieved content (RFC 0002). This is the injection containment, and it sits before the human, so an approver is never shown a plausible-looking preview manufactured by a document.
5. **Pending actions expire.** Default 72 hours. An expired action is not resumable, because the material it was based on may have changed.
6. **Rejection is final for that run.** The run is abandoned. A member who still wants it asks again, and gets a fresh preview.

### Idempotency

The Agent generates an idempotency key at preview and reuses it on every execution attempt for that action, including after a resume or a retry. A provider must make repeated `execute` calls with the same key produce one effect, by one of three means declared in its classification (RFC 0004):

- `key`: the upstream accepts an idempotency key. Pass it through. Preferred.
- `natural`: the effect is naturally idempotent, such as setting a field to a value, or the upstream deduplicates. Document why.
- `no`: neither. Allowed only for reversible classes, and the provider must record the key and its outcome itself so a repeat is detected locally.

`idempotent: no` with `reversible: false` is rejected at startup (RFC 0004). An irreversible effect that cannot be deduplicated cannot be retried, and a partial failure would leave no safe move.

### Blast radius

Approval is a check on intent, not on magnitude; an approver clicking through a queue will approve a broadcast to the whole organization as readily as one to a committee. Each provider declares bounds that the system enforces independently:

```python
class Limits(BaseModel):
    max_scope: int               # recipients, rooms, repositories per action
    max_per_hour: int            # executions per actor
    max_scope_per_day: int       # cumulative scope per actor
    requires_second_approver_above: int | None
```

- An action whose `scope` exceeds `max_scope` is refused, not escalated. Doing it means splitting it deliberately.
- An action over `requires_second_approver_above` needs a second approver, which is how a broadcast to a committee and a broadcast to everyone are treated differently without two providers.
- Rate limits are per actor and per class, counted in the Surface against the audit log.
- **A kill switch per class and per provider.** Setting it refuses previews and executions immediately, without a deploy, and the reason is surfaced in the UI. `broadcast` ships with it, because the first thing anyone will want during an incident is for the mail to stop.
- **Dry run in every non-production profile.** In the `dev`, `preview`, and `staging` secretspec profiles, every provider defaults to an implementation that records what it would have done and changes nothing (RFC 0004). Reaching a real upstream requires the `prod` profile. A preview deployment must never be able to email the organization.

### Audit

Every transition is recorded in the Surface's append-only `audit_log` (RFC 0003): previewed, approved, rejected, expired, executed, failed. A record carries the actor, the action id, the capability class, the full arguments, the preview hash, the approver, the idempotency key, the result or error, and timestamps.

Arguments are retained, which splits bark RFC 0004's rule that tool arguments are never logged. The Bark reasoning is that arguments quote member messages and so are personal data. It holds for reads, where the argument is a search query and nothing depends on reconstructing it, and the read path keeps the rule. It does not hold for writes: an action nobody can reconstruct is an action nobody is accountable for, and accountability is the reason the approval exists. Access is gated on `audit.read`, retention is finite and configurable, and the record of the approval itself is kept longer than the arguments where that distinction is useful.

No provider may have a path that reaches `execute` without a record, so the record is opened before the effect and completed after it. An execution that crashes in between leaves a record in an indeterminate state, which is the correct outcome: it needs a human to reconcile, and silence would not.

## Alternatives Considered

- **Building room booking and mass emailing directly, without this contract.** It is the shortest path to the two features the team named, and would ship sooner. Rejected because the hard parts are shared, the second feature would duplicate the first, and the irreversible one would be the one built in a hurry. The cost of this decision is real: no user-visible feature ships from this RFC alone.
- **One generic `action` class with per-provider policy.** Fewer concepts, and every provider states its own rules. Rejected: policy would then live in as many places as there are providers, and the review question "what can a broadcast do" would have no single answer. Four classes mean four policies to review and a new provider inherits one.
- **Approving at the conversation level rather than per action.** A member approves the plan once and the agent carries it out. Rejected: the plan's steps are generated by a model from retrieved content, so approving a plan approves arguments that do not exist yet. This is precisely the shape an injection would exploit.
- **Trusting the Agent's permission check and skipping the server's.** One check, less to keep in sync. Rejected for the reason in RFC 0004: the case that matters is an Agent that was induced to call a tool it should not have bound.
- **Letting `preview` perform a reversible version of the effect**, such as a tentative hold or a draft, where the upstream has no read-only check. It would make more upstreams implementable. Rejected as the default because a preview with effects is not a preview and the distinction would erode. A provider that genuinely needs a tentative hold models it as two actions, a reversible `reservation` to hold and a second to confirm, each with its own approval.
- **Auto-approving low-risk actions.** It is tempting for `record`, and is permitted there by configuration. Rejected as a general mechanism, and never available for `broadcast`, because the threshold for "low risk" drifts and the audit value of a human decision is most of the point.
- **Keeping bark RFC 0004's rule that arguments are never logged.** It is the stronger privacy position and it is Bark's considered decision. Rejected for writes only, with the reasoning above. The read path is unchanged.

## Open Questions

- Who are the approvers in practice, and does `action.approve.broadcast` map to team leads, to a dedicated role, or to a named list? This is a governance question as much as a technical one.
- Should an approval be allowed to amend arguments, within limits, rather than only approve or reject? It would avoid a round trip for a typo in a draft, at the cost of making the approved bytes a thing the approver can edit. Proposed: no for now, and the member re-asks.
- Does `reservation` need a hold-then-confirm pair from the outset, or does that depend on the first booking upstream's API?
- What is the right retention period for write arguments, given that the approval record should outlive them?
- Does the kill switch need to interrupt an execution already in flight, or only refuse new ones? Only refusing new ones is proposed, since providers are expected to be short-running.

## Implementation Phases

**Contract**

- `ActionProvider`, `Preview`, `Limits`, the four classes, and the registry on `openbark-actions`
- Argument hashing, idempotency key generation and reuse, approval reference checking on both sides
- Dry-run provider implementations and the per-class kill switch
- `actions` and `audit_log` in the Surface, the approval queue, inline previews, approve and reject (RFC 0003)
- Run suspension and resume in the Agent (RFC 0002)

**First provider: `record`**

- Reminders, task lists, and events, reversible and internal, as the end-to-end exercise of the lifecycle
- Adversarial evaluations from RFC 0002, which gate enabling any write tool in production

**Then, in increasing order of consequence**

- `change_request` over git.cmu.dev
- `reservation`, once a booking upstream with a real availability check is identified
- `broadcast`, last, with four eyes, limits, and the kill switch exercised before it is enabled
