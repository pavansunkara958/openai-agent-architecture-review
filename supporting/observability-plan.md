# Tracing and Observability Plan

Detail behind §6 of the review.

---

## Two sinks, on purpose

| Sink | Carries | Used for |
|---|---|---|
| **OpenAI traces dashboard** | SDK spans, grouped by `group_id = ticket_id` | investigating one ticket |
| **OTLP → Tempo + Prometheus** | spans, metrics, alerts | alerting, retention, correlation with infra |

`trace(workflow_name="internal-support", group_id=ticket_id)` groups a whole
multi-turn ticket into one inspectable unit rather than scattered runs. A
ticket that spans an approval wait of two days is still one trace.

The second sink exists because an investigation that requires a vendor
dashboard login is not evidence the customer's security team can work with, and
because dashboard retention is not under their control.

---

## Spans

| Span | Key attributes |
|---|---|
| `support.ticket` (root) | `ticket.id`, `.category`, `.outcome`, `requester.hash`, `agent.routed_to` |
| `gen_ai.chat` | `gen_ai.request.model`, input/output tokens, derived cost, purpose |
| `agent.handoff` | `from`, `to`, `depth` |
| `agent.tool` | `tool.name`, `.outcome` (`ok`/`refused`/`error`), `.refusal_reason`, `.args_hash` |
| `agent.guardrail` | `layer`, `decision` (**allow or block**), `reason`, `false_positive` |
| `agent.approval` | `action`, `approver.hash`, `token.jti`, `verification_result` |

Three conventions that matter more than they look:

**Guardrail spans record `allow` as well as `block`.** Recording only blocks
gives a denominator of zero, and the block *rate* is the signal — a raw count
rises with ticket volume and means nothing.

**`tool.outcome` separates `refused` from `error`.** A privileged group
membership refused pending approval is the control working. Folding it into an
error rate fakes an incident and hides a real one in the noise.

**Requester identity and tool arguments are hashed, never stored.** The hash is
enough to correlate a retry loop or group one person's tickets; it is not
enough to turn the trace store into an HR data source. Traces are long-retained
and widely readable — the decision belongs there, the payload does not.

---

## Metrics

| Metric | Labels |
|---|---|
| `support.tickets` | `category`, `outcome` |
| `support.ticket.duration_ms` | `outcome` |
| `agent.handoffs` | `to`, `depth` |
| `agent.tool.calls` | `tool`, `outcome` |
| `agent.guardrail.decisions` | `layer`, `decision` |
| `agent.guardrail.false_positives` | `layer` |
| `agent.approvals` | `action`, `result` |
| `agent.cost_usd` | `model` |

Emitted at the point of measurement, not derived from spans. Metrics survive
sampling; spans carry the detail that explains a metric once it moves.
Deriving cost only from sampled spans reports a fraction of real spend.

---

## Alerts

| # | Alert | Class | Severity | Threshold |
|---|---|---|---|---|
| 1 | Ticket error rate | RED | P2 | > 3%, 10 min |
| 2 | p95 ticket latency | RED | P3 | > 20s, 2 periods |
| 3 | Cost per ticket | RED | P3 | > 2.5× 7-day baseline |
| 4 | **Guardrail false-positive rate** | Behavioural | **P2** | > 8% of blocks |
| 5 | **Routing-mix drift** | Behavioural | P2 | TVD > 0.15 vs baseline |
| 6 | Tool refusal-rate change | Behavioural | P3 | > 2× baseline |
| 7 | **Approval token verification failure** | **Security** | **P1** | any |
| 8 | Approval queue age | Operational | P3 | oldest > 48h |
| 9 | Handoff depth at cap | Behavioural | P3 | > 5% of tickets |

### The two worth defending

**#4 and #5 are invisible to RED monitoring.** A regressed guardrail or a
misrouting router makes every RED metric look *better* — blocked and misrouted
tickets skip tools and finish faster, so latency drops and cost falls while
employees stop getting help. Without the behavioural class, that failure is
found when someone complains.

**#7 should be impossible.** The token is signed, parameter-bound and
single-use, so a verification failure means either a bug or an attempt to forge
one. Both warrant waking someone. A control nobody has ever seen fail is a
control nobody has tested.

---

## Dashboard

Four sections, ordered by the question being asked:

1. **Is it up?** — volume, error rate, p50/p95/p99
2. **What is it costing?** — tokens, cost per ticket, spend by category
3. **Is it behaving?** — routing mix, guardrail blocks by reason, false-positive
   rate, tool outcomes, approval queue
4. **What do I open first?** — longest tickets, failures, trace ids

Section 3 is the one a standard APM dashboard does not have.

---

## Verifying the observability itself

A monitoring system fails by reporting green: a metric never emitted shows as
zero, and zero looks like health.

- **Every alert is fire-drilled before go-live** — triggered deliberately,
  confirmed to page the right rota. An alert that has never fired is a
  hypothesis.
- **Span emission is asserted in CI.** A build producing no telemetry fails
  the gate.
- **Attribute names are unit-tested against dashboard queries**, so a rename
  breaks the build instead of silently zeroing a panel.
