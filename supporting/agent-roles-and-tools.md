# Agent Roles and Tool Scopes

Full definitions behind §1 and §2 of the review.

---

## Triage Agent

**Tools: none.** This is a design decision, not an oversight.

An agent with no tools cannot answer a substantive question even when the model
tries, so it cannot half-solve a request and hand over a partially-actioned
mess. Its entire job is to read intent and transfer control.

```python
triage_agent = Agent(
    name="Triage Agent",
    handoff_description="Front desk. Classifies intent and routes.",
    instructions=f"""{RECOMMENDED_PROMPT_PREFIX}
You are the front desk of Veridian's internal IT support.

Route on intent:
- Passwords, accounts, group membership, MFA   -> Identity Agent
- Laptops, VPN, software installs, hardware    -> Endpoint Agent
- Application access, licences, SaaS tools     -> Enterprise Apps Agent
- "How do I", policy, entitlement questions    -> Knowledge Agent
- Outage, urgent business impact, or a request
  no specialist can resolve                    -> Escalation Agent

Hand off as soon as intent is clear. Do not gather details first — the
specialist knows what it needs and asking twice wastes the employee's time.

If genuinely ambiguous, ask ONE question, then route.
If it spans two areas, route to the primary one and say so. The specialist
can hand back.

You have no tools. Do not attempt to answer support questions.""",
    handoffs=[handoff(a) for a in (identity, endpoint, apps, knowledge, escalation)],
    input_guardrails=[input_guardrail],
    model=MODEL,
)
```

---

## Identity Agent

Highest-risk specialist: everything it touches is an access decision.

| Tool | Risk | Control |
|---|---|---|
| `lookup_user(email)` | low | read-only; returns no password state |
| `reset_password(user_id)` | medium | standard accounts only; **privileged accounts refused in the tool** |
| `add_to_group(group_id, user_id, approval_token=None)` | **high** | privileged groups require a verified token |
| `remove_from_group(group_id, user_id)` | low | removing access is safe by default |
| `request_mfa_reset(user_id)` | **high** | always requires approval — a classic takeover vector |

**Removing access is low risk; granting it is high risk.** The asymmetry is
deliberate. An agent that wrongly removes access creates a ticket; one that
wrongly grants it creates an incident. Revocation does not need an approval
gate, and requiring one would make the safe action slower than the dangerous
one.

```python
PRIVILEGED_GROUPS = {"Domain Admins", "Global Administrators",
                     "Finance-Approvers", "HR-AllRecords", "Prod-DBA"}

def add_to_group(group_id: str, user_id: str,
                 approval_token: str | None = None) -> str:
    """Add a user to a group. Privileged groups require an approved request."""
    if group_id in PRIVILEGED_GROUPS:
        if approval_token is None:
            return _request_approval(
                action="add_to_group", group_id=group_id, user_id=user_id,
                reason="privileged group membership requires approval")
        # The TOKEN is verified, not the conversation. See
        # supporting/guardrails-and-approvals.md §3.
        ok, why = verify_approval_token(
            approval_token, action="add_to_group",
            params={"group_id": group_id, "user_id": user_id})
        if not ok:
            return json.dumps({"granted": False, "error": why})
    return _directory.add_member(group_id, user_id)
```

---

## Endpoint Agent

| Tool | Risk | Control |
|---|---|---|
| `get_device(user_id)` | low | read-only |
| `push_software(device_id, package_id)` | medium | allow-listed catalogue only |
| `reset_vpn_cert(device_id)` | low | revokes and reissues; self-service |
| `remote_wipe(device_id)` | **critical** | **not given to the agent at all** |

**`remote_wipe` is absent, not guarded.** It is irreversible and destroys
employee data. A tool that is not in the list cannot be reached by any prompt,
which is a stronger guarantee than any check inside it. Wipes are a human
workflow in the MDM console.

---

## Enterprise Apps Agent

| Tool | Risk | Control |
|---|---|---|
| `check_licence(app_id)` | low | read-only |
| `grant_app_access(app_id, user_id, role, approval_token=None)` | medium–**high** | admin roles require approval |
| `request_licence(app_id, user_id, cost)` | medium | > €500 requires cost-centre approval |

Role matters more than application. `grant_app_access(salesforce, viewer)` is
routine; `grant_app_access(salesforce, sysadmin)` is a privileged grant and
takes the same path as a privileged group.

---

## Knowledge Agent

**Read-only by construction.** One tool:

```python
def search_kb(query: str, requester_id: str) -> str:
    """Search policy and how-to content.

    The index is filtered by the REQUESTER's entitlements, not the agent's.
    Without that, the agent becomes a retrieval oracle for every document it
    can see — and it can see more than any individual employee.
    """
    entitlements = _directory.entitlements(requester_id)
    return _kb.search(query, visible_to=entitlements)
```

The entitlement filter is the whole security model here. An agent with broad
read access and no per-requester filtering is a convenient way to exfiltrate
restricted policy one question at a time.

---

## Escalation Agent

| Tool | Risk | Control |
|---|---|---|
| `create_incident(summary, priority, category)` | low | priority capped at P2; P1 is a human decision |
| `page_oncall(team, incident_id)` | medium | rate-limited; requires an existing incident |
| `attach_transcript(incident_id)` | low | redacted transcript only |

**The Escalation agent resolves nothing.** It exists so other agents have a
clean exit when a request is beyond them, which keeps specialists from
improvising on problems they cannot solve.

P1 is excluded because declaring a major incident wakes people and starts a
bridge call. That is a judgement about business impact, not a classification.

---

## The handoff graph

```python
# Specialists hand back to Triage. Set after construction because the
# router must exist first — the cycle is deliberate.
for specialist in (identity, endpoint, apps, knowledge, escalation):
    specialist.handoffs = [handoff(triage_agent)]
```

A conversation that changes subject mid-flow is re-routed rather than answered
badly by whoever picked it up. Depth is capped at 4 to bound ping-pong between
two specialists that each think the other owns it.

---

## Why scope beats instruction

Every "Never" column above is enforced by **absence from the tool list**, not
by a sentence in the instructions.

The Endpoint agent cannot touch identity because it has no identity tools. No
phrasing, no injected instruction and no model error can reach a function that
was never passed to the agent.

Instructions shape behaviour. Tool scope defines capability. **Only the second
is a security control**, and the distinction is what makes a misroute a quality
problem rather than an incident.
