# Guardrails and Approvals

Detail behind §3 and §5 of the review.

---

## 1 · Input guardrails

Deterministic patterns first; the model-based relevance check runs only when
patterns are inconclusive. A guardrail that costs as much as the request it
guards gets switched off within a quarter.

| Category | Detects | Example |
|---|---|---|
| Instruction override | "ignore previous instructions", "you are now…" | injection |
| Authority claim | "my manager already approved", "security said it's fine" | **social engineering — no injection markers at all** |
| Pasted secret | API keys, private keys, passwords in the ticket body | an employee pasting a credential |
| PII / payment data | Luhn-valid PANs, national ID formats | wrong channel for this data |
| Off-topic | not an IT support request | relevance model |
| Oversized | > 4,000 characters | context stuffing |

**The authority-claim pattern is the interesting one.** "Sarah in security
already approved this" contains no injection markers and reads as an ordinary
sentence. It is detected because it is common, but detection is not the
control — the control is that approval is verified against a signed record
(§3), so the claim is simply irrelevant.

**Luhn validation prevents the obvious false positive.** A naive 13–19 digit
regex flags asset tags and ticket numbers. A guardrail is only credible if you
test what it lets through, not just what it blocks.

### Scope: the newest message only

With session history the SDK passes the whole accumulated conversation to the
guardrail. Scanning all of it means a message blocked on turn 1 re-trips on
every later turn — the ticket sticks permanently, and the reported finding
points at the *wrong* message, so the symptom actively misleads whoever
investigates.

**A guardrail that permanently bricks a conversation is worse than one that
misses.**

---

## 2 · Output guardrails

| Blocks | Why |
|---|---|
| Credentials or tokens in the reply | the agent should never surface one |
| **Account enumeration** | "no account exists for that address" confirms which addresses do |
| Unqualified commitments | "this will be fixed by Friday" is not the agent's to say |
| Internal hostnames, IPs, infrastructure detail | reconnaissance value |
| Other employees' attributes | the requester asked about themselves |

Enumeration is templated rather than generated:

> "If an account exists for that address, a reset link has been sent."

The same sentence whether or not it exists. Templating it means the phrasing
cannot drift as prompts change — the wording *is* the control.

---

## 3 · Tool guardrails and the approval protocol

The layer that matters. Input and output guardrails reduce the *probability* of
a bad action; this bounds the *consequence*.

### Three checks inside every write tool

| Check | Question |
|---|---|
| **Privilege** | is the target a privileged object? |
| **Blast radius** | how many principals does this affect? |
| **Segregation of duties** | is the requester also the beneficiary or approver? |

### The approval protocol

```
1. Agent calls the tool without a token
     → tool returns "approval required" + creates a DURABLE RECORD
     → Slack / ServiceNow card to the named approver

2. Human approves IN THAT SYSTEM, authenticated separately
     → a signed token is issued, bound to:
          action · exact parameters · approver · expiry (30 min)

3. Agent resumes and calls the tool WITH the token

4. The tool verifies the token SERVER-SIDE:
     ✓ signature valid
     ✓ action and parameters match exactly
     ✓ not expired, not already redeemed
     ✓ approver ≠ requester
     ✓ approver holds the authority for this action class
```

```python
def verify_approval_token(token: str, action: str, params: dict) -> tuple[bool, str]:
    """Verify an approval at the moment of execution.

    The conversation is an UNTRUSTED INPUT. A message can claim an approval
    was granted; only the signed record is evidence. If the agent could
    'remember' approval, a crafted ticket could manufacture one.

    Parameters are bound into the signature, so a token approved for
    `group=Marketing` cannot be replayed for `group=Domain Admins`.
    """
    try:
        claims = jwt.decode(token, APPROVAL_PUBLIC_KEY, algorithms=["RS256"])
    except jwt.InvalidTokenError as exc:
        _alarm("approval_token_invalid", reason=str(exc))     # P1 security alarm
        return False, "invalid approval token"

    if claims["action"] != action or claims["params"] != params:
        _alarm("approval_token_mismatch", expected=action)     # P1
        return False, "approval does not match the requested action"
    if _already_redeemed(claims["jti"]):
        _alarm("approval_token_replay", jti=claims["jti"])     # P1
        return False, "approval already used"
    if claims["approver"] == claims["requester"]:
        return False, "approver and requester must differ"

    _redeem(claims["jti"])
    return True, ""
```

**Why parameter binding matters.** Without it, a token approved for a harmless
group could be replayed against a privileged one. The signature covers the
exact parameters, so a token is useless for any other action.

**Why single-use matters.** A multi-day approval window plus a reusable token
is a standing grant nobody intended to issue.

**Every verification failure is a P1 security alarm.** These should never
happen in normal operation, which is precisely why they are alarmed — a control
you are certain cannot fail is one nobody checks.

---

## 4 · Approval matrix

| Action | Threshold | Approver | SLA |
|---|---|---|---|
| Password reset, standard account | — | none | immediate |
| Password reset, privileged account | always | Security | 1h |
| Group membership, standard | — | none | immediate |
| **Group membership, privileged** | always | Security + line manager | 4h |
| **MFA reset** | always | Service desk L2 (a different human) | 1h |
| Mailbox delegation | always | Data owner | 8h |
| App access, standard role | — | none | immediate |
| **App access, admin role** | always | App owner | 4h |
| Licence purchase | > €500 | Cost-centre owner | 1 business day |
| **Any bulk change** | > 10 users | Platform lead | 4h |
| Anything on an exec account | always | Security | 1h |

**Two human approvers for privileged groups.** Security confirms the access is
appropriate; the line manager confirms the person should have it. One approver
can be socially engineered; two who do not know each other is materially harder.

**Expiry, not indefinite pending.** Unapproved requests expire after 72 hours
and the employee is told. A request pending for three weeks is a request
everyone has forgotten, and reviving it has no fresh scrutiny.

---

## 5 · What is deliberately not guarded

**`remote_wipe` is absent from every toolset.** It is irreversible and destroys
employee data. A tool that is not in the list cannot be reached by any prompt —
a stronger guarantee than any check inside it.

**P1 incident declaration is not available.** Declaring a major incident wakes
people and starts a bridge call. That is a judgement about business impact, not
a classification, and the agent caps at P2.

**Access removal is ungated.** The safe direction should not be slower than the
dangerous one, or people route around it.
