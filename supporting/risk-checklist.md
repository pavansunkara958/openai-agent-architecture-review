# Risk Analysis and Mitigation Checklist

Detail behind §7 of the review. Each item names where the control lives and how
it is verified — "we use guardrails" is not a reviewable claim.

---

## Security

| # | Risk | Impact | Control | Lives in | Verified by |
|---|---|---|---|---|---|
| S1 | Injection → privileged group grant | **Critical** | privileged-group check + approval token | `tools.py` | unit test; CI scope assertion |
| S2 | Approval claimed in the transcript | **Critical** | token verified server-side, never the model | `verify_approval_token` | negative test: forged token must fail |
| S3 | Token replayed on a different target | High | parameters bound into the signature | token claims | negative test |
| S4 | Token reused | High | single-use `jti` | redemption table | negative test |
| S5 | Requester approves their own request | High | approver ≠ requester | token verification | unit test |
| S6 | Account enumeration | Medium | templated non-committal response | output guardrail | assertion on exact phrasing |
| S7 | Knowledge agent surfaces restricted policy | Medium | KB index filtered by **requester** entitlements | `search_kb` | per-entitlement test fixtures |
| S8 | Tool scope widened by a refactor | High | per-agent scope asserted in CI | test suite | fails the build |
| S9 | Social engineering with no injection markers | High | **tool-layer approval, not pattern detection** | `tools.py` | eval case A04-equivalent |
| S10 | Credential pasted into a ticket | Medium | input guardrail; redact before logging | guardrail + tracing | sampled log scan |
| S11 | Trace store becomes an HR data source | Medium | requester id and tool args hashed | `tracing.py` | attribute-name test |

**S9 is the one patterns cannot solve.** "My manager already approved this"
reads as an ordinary sentence. It is detected because it is common, but
detection is not the control — the control is that the claim is irrelevant,
because approval is a signed record the model cannot produce.

---

## Operational

| # | Risk | Impact | Control | Verified by |
|---|---|---|---|---|
| O1 | Silent routing regression | High | routing-mix alarm; eval suite gates deploys | alert #5; CI |
| O2 | Guardrail widened → legitimate tickets blocked | High | false-positive rate alarm; FP test required in the same PR | alert #4; PR check |
| O3 | Model version drift | Medium | pinned version; eval before upgrade | deploy gate |
| O4 | Handoff ping-pong | Low | depth cap 4; alarm above 5% at cap | alert #9 |
| O5 | Approval queue stalls | Medium | 72h expiry; queue-age alarm | alert #8 |
| O6 | Cost runaway | Medium | per-ticket budget; tiered models | alert #3 |
| O7 | Session state lost mid-approval | **High** | **approvals are durable records, not session state** | restart test |
| O8 | Dependency outage (directory, MDM) | Medium | graceful degradation to Escalation agent | fault-injection drill |

**O7 is a design decision rather than a mitigation.** A pending approval cannot
live in conversation state because an approver may act days later. It is a
database row; the agent resumes from it. Verified by restarting the service
mid-approval and confirming the ticket completes.

---

## Pre-launch checklist

Not recommendations. The platform does not go live until each is true.

- [ ] Every write tool refuses without a valid approval token where the matrix
      requires one — tested with a **forged** token, not just a missing one
- [ ] Tool scope per agent asserted in CI; widening requires security review
- [ ] Guardrail false-positive rate ≤ 2% on the evaluation set
- [ ] Evaluation suite includes adversarial cases with **no lexical markers**
- [ ] Enumeration phrasing identical for existing and non-existing accounts
- [ ] Knowledge index filtered by requester entitlements, with fixtures proving it
- [ ] All nine alerts fire-drilled and routed to a rota that is on call
- [ ] Approval expiry (72h) and token expiry (30 min) both tested
- [ ] Approver ≠ requester enforced and tested
- [ ] Model version pinned; eval suite green against that exact version
- [ ] Traces contain no raw requester identity or tool arguments — sampled scan
- [ ] Restart-mid-approval test passes

The forged-token test and the no-marker adversarial cases are the two that get
skipped. They are also the two that distinguish a control that works from a
control that has only ever been pointed at friendly input.

---

## Residual risk

Stated because a risk section listing none has not been written honestly.

| Residual | Why it remains | Bounded by |
|---|---|---|
| Prompt injection | open problem; no filter is complete | privileged actions need a signed approval the model cannot forge — worst case is a *request* a human declines |
| Knowledge over-disclosure within entitlements | the agent answers faster than a human would look things up | entitlement filter; accepted by the information-security owner |
| Approver fatigue → rubber-stamping | human factor, not technical | two approvers for privileged grants; monthly sampling of approvals |
| Model behavioural drift between pinned versions | provider-side | eval suite before every upgrade |

**Approver fatigue is the one with no technical fix.** If approvals become
routine they stop being scrutiny. The mitigation is volume discipline — keeping
the approval matrix narrow enough that an approval remains a notable event — and
monthly sampling by Security.
