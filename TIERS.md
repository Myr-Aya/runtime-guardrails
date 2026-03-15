# Agent Guardrails — Risk Tiers and Controls

## Universal control table

Controls depend only on the final tier and the guardrails mode. They are the same for every action at that tier.

### Tier 1 — Routine (Green)

Not applicable. Tier 1 actions are excluded from the gate and do not trigger the gate process.

### Tier 2 — Standard (Amber)

Modifies persistent local state or involves authenticated external reads.

**Full gate:** Notify the user (briefly state the action and its tier classification). Proceed unless the user objects. Log.

**Audit-only:** Auto-proceed. Log with action detail.

### Tier 3 — Elevated (Red)

Involves outbound data transmission, communication on behalf of the user, credential handling, or modifications to systems outside the local workspace.

**Full gate:** Pause. Present the action, its scope, and what data is involved. Wait for explicit user approval. If denied, log denial and stop.

**Audit-only:** Auto-proceed. Alert the user via their configured channel with full details of the action taken. Log.

### Tier 4 — Critical (Black)

Irreversible actions with significant consequences. Financial impact, bulk destruction, permission changes, public-facing effects, or attack surface modifications.

**Full gate:** Full stop. Warn the user concisely: state the action, what is at risk, and whether it is reversible. Ask for approval. If the user requests details, provide a full impact assessment (what will happen, what cannot be undone, blast radius). If the user gives explicit approval, execute and log. If denied or ambiguous, log denial and stop.

**Audit-only:** HALT. Do NOT execute. Alert the user immediately via their configured channel with full details of the attempted action. Log with "HALTED" status. Only `GUARDRAILS_TIER4_AUTONOMOUS=alert-and-execute` overrides this (not recommended).

## Escalation rules

Escalate an action by one tier (e.g. Tier 2 becomes Tier 3) when ANY of these conditions apply:

- **Unfamiliar target:** the action targets a system, endpoint, or file the agent has not interacted with in this session.
- **Third-party data:** the action involves data belonging to someone other than the current user.
- **Scope expansion:** the agent plans to do more than the user explicitly requested (e.g. user asked to edit one file, agent plans to edit three).
- **Bulk operation:** the action affects 5 or more files, records, or targets.
- **Chained actions:** the action is part of a chain of 3+ sequential sensitive actions. Escalate the entire chain, not just individual steps.
- **Destination change:** the action targets a different recipient, endpoint, folder, or publication surface than what the user explicitly indicated or from prior session context.

Tier 4 is the maximum — no further escalation. Apply the Tier 4 control.

## De-escalation

- **Relaxed mode** (`/guardrails relaxed`): Tier 2 actions skip the gate entirely (treated as Tier 1). Tier 3 and 4 are never de-escalated.
- **Pre-approved patterns** (`GUARDRAILS_PRE_APPROVED`): matching actions skip approval and alerting, but still undergo classification, escalation, and hard-constraint checks. Log with the tier and a "PRE-APPROVED" tag. If escalation raises the action to Tier 4, pre-approval is void.
- De-escalation never applies to Tier 4 actions. Tier 4 controls are absolute.
