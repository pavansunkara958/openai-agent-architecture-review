# OpenAI Agent Architecture Review

**Enterprise internal support platform** — Veridian Group, 28,000 employees,
~4,200 IT tickets/week. Built on the OpenAI Agents SDK and Responses API.

## → [**review.md**](review.md) ←

The design review document. Everything the brief asks for:

| § | Covers |
|---|---|
| 1 | System architecture diagram and agent role definitions |
| 2 | Handoff strategy and routing decision logic |
| 3 | Input, output and tool guardrails, and how each is enforced |
| 4 | Stateful workflow — SDK sessions and the Responses API |
| 5 | Human-in-the-loop approval points for high-risk actions |
| 6 | Tracing and observability plan |
| 7 | Risk analysis and mitigation |

## Supporting artifacts

| Document | Contents |
|---|---|
| [agent-roles-and-tools.md](supporting/agent-roles-and-tools.md) | Six agents, per-tool risk ratings, code for the handoff graph and privileged-group check |
| [guardrails-and-approvals.md](supporting/guardrails-and-approvals.md) | Guardrail catalogue, the approval protocol, token verification, the approval matrix |
| [observability-plan.md](supporting/observability-plan.md) | Span and metric inventory, nine alerts, dashboard layout |
| [risk-checklist.md](supporting/risk-checklist.md) | 19 risks with controls and verification, plus a pre-launch checklist |

---

## The two decisions this architecture rests on

### Approval is verified by the tool, never asserted in the conversation

The agent can grant privileged access, which makes it a target. If it could
*remember* that a human approved something, a crafted ticket could claim an
approval happened — and "my manager already signed off" contains no injection
markers and reads as an ordinary sentence.

So approval is a **signed, parameter-bound, single-use token** issued out of
band when a human approves in Slack or ServiceNow. The tool verifies it at the
moment of execution: signature, exact parameters, expiry, not-already-redeemed,
and approver ≠ requester.

**The transcript is an untrusted input. The signed record is the fact.** Every
verification failure is a P1 security alarm, because it should never happen.

### A pending approval cannot live in conversation state

An approver may act tomorrow, or next week. If the pending change sat in
session state, a session expiry or a redeploy would silently lose it and the
employee would be waiting on something that no longer exists.

So the architecture separates two kinds of state: **SDK sessions** carry the
conversation; a **durable record** carries the approval workflow, and the agent
resumes from it. The Responses API threads the post-approval continuation
server-side — the right tool for a single-shot step with no tools and no
handoffs.

---

## What it does not claim

**Prompt injection is not solved**, and saying otherwise would be dishonest.
Guardrails lower the probability that a crafted ticket reaches a tool with bad
instructions. What bounds the outcome is that privileged actions require a
signed approval the model cannot forge — so the worst case is a *request* a
human declines, not an escalation.

**Probability controls and consequence controls are different things, and only
the second survives an attacker cleverer than your patterns.**

Two tools are absent rather than guarded: `remote_wipe`, because it is
irreversible, and P1 incident declaration, because waking people is a judgement
about business impact. **A tool that is not in the list cannot be reached by any
prompt** — a stronger guarantee than any check inside it.
