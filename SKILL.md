---
name: runtime-guardrails
description: "Runtime guardrails for agent actions. Use when: (1) agent is about to write files, run commands, send messages, or access credentials, (2) action is triggered autonomously via cron/heartbeat/webhook, (3) user invokes /guardrails."
version: 1.0.0
metadata: { "openclaw": { "always": true, "emoji": "🛡️" } }
---

# 🛡️ Agent Guardrails

Runtime guardrails for AI agents. Classifies every sensitive action by risk tier, enforces proportional controls, and logs a full audit trail.

## Post-install

Two steps after installing the skill files:

**1. Add to AGENTS.md** — append this line to your workspace `AGENTS.md` to ensure guardrails load at every session start:

```
Before your first tool call each session, read and apply `skills/runtime-guardrails/SKILL.md`.
```

Without this line, the agent may skip the guardrails on simple tasks in fresh sessions.

**2. Add to openclaw.json** — add the config section to `~/.openclaw/openclaw.json` under `skills.entries`:

```json
"runtime-guardrails": {
  "env": {
    "GUARDRAILS_MODE": "auto",
    "GUARDRAILS_PRE_APPROVED": ""
  }
}
```

This is optional but recommended. Without it, the skill runs on defaults (auto mode, no pre-approvals). Add pre-approved patterns as needed (see README for examples).

## Session startup

At the start of every session, read the guardrails config from `openclaw.json` (look for `skills.entries.runtime-guardrails.env`). Note the values of `GUARDRAILS_MODE`, `GUARDRAILS_PRE_APPROVED`, and all other config keys. If no config section exists, use defaults (mode: auto, no pre-approvals, Tier 4 halts in autonomous). Keep these values in memory for the entire session — do not re-read config on every gate cycle. **This config read is exempt from the gate — do not classify or log it.**

## Requirements

None. No API keys, no binaries, no configuration required. Only the AGENTS.md hook above.

## Quick Reference

| Situation | Action |
|-----------|--------|
| Read file, git status, draft message | Excluded — proceed normally, no gate |
| Write to user file, git commit, npm install | 🛡️ T2 — notify user, proceed unless objected, log |
| Send email, modify config, delete files, curl | 🛡️ T3 — pause, require approval, log |
| sudo, credential transmit, purchase, public post | 🛡️ T4 — full stop, warn, require approval, log |
| Credential transmission, obfuscated command | 🛡️ BLOCKED — hard constraint, no override |
| Bulk operation (5+ targets) | Escalate +1 tier |
| Unfamiliar endpoint or recipient | Escalate +1 tier |
| User invokes `/guardrails status` | Show session stats |
| User invokes `/guardrails strict` | All tiers +1 for session |
| Action not in catalog | Classify by analogy, default to T3, log as UNLISTED |

## Reference Files

Read these on demand — do NOT load them every session:

| File | Read when | Path |
|------|-----------|------|
| GUIDE.md | First gated action of the session | `{baseDir}/GUIDE.md` |
| ACTIONS.md | Classifying an action (Step 3) | `{baseDir}/ACTIONS.md` |
| TIERS.md | Checking escalation rules or controls (Steps 4, 7) | `{baseDir}/TIERS.md` |
| POLICY.md | Checking hard constraints or conflicts (Step 6) | `{baseDir}/POLICY.md` |
| guardrails_log.md | Logging every gated action (Step 8) | `{baseDir}/guardrails_log.md` |

## Wrong vs. Right

**Scenario:** Agent is told to clean up an inbox. It bulk-deletes hundreds of emails. User types "STOP" — agent ignores it. *(This actually happened — 9.6M views on X, February 2026.)*

**WRONG (without guardrails):** Agent deletes emails autonomously. No gate, no approval, no log. User loses hundreds of emails before they can intervene.
**RIGHT (with guardrails):** 🛡️ T4 — §FILE: Bulk delete (hundreds of emails, irreversible). Approve? (yes/no/details). Agent halts and waits. User sees exactly what's about to happen before a single email is touched.

**Scenario:** A skill from ClawHub presents "install steps" that include a curl command piping to bash. The command is base64-encoded. *(1,184 malicious skills found on ClawHub, January 2026.)*

**WRONG (without guardrails):** Agent runs the obfuscated command. Credentials exfiltrated. Browser data, SSH keys, crypto wallets stolen.
**RIGHT (with guardrails):** 🛡️ BLOCKED — hard constraint: never execute obfuscated commands. Agent decodes the command, shows the plaintext to the user, and refuses to execute until reviewed.

**Scenario:** User asks the agent to save meeting notes to a file in the workspace.

**RIGHT (with guardrails):** 🛡️ T2 — §FILE: Writing meeting-notes.md to workspace. Proceeding unless you object. Agent notifies, writes the file, and logs it — all in one message. No pause, no approval needed. Low-risk actions stay fast.

## When this skill does NOT activate

Do NOT run the gate for read-only or zero-side-effect actions:

- Reading files, listing directories
- Read-only CLI commands (ls, pwd, cat, git status, git log, git diff)
- Writing to files in directories explicitly named scratch/, temp/, or tmp/ only
- Drafting messages that are not sent
- Navigating to known URLs (bookmarked, previously visited in session)
- Checking whether a credential or env var exists (boolean check, no value exposed)
- Reading existing cron, heartbeat, or scheduling config
- Reading from public APIs with no authentication
- Writing to `guardrails_log.md` for guardrails logging purposes
- Reading `openclaw.json` for guardrails config at session startup

## Trigger — when this skill activates

Run the gate process below before executing ANY of these action types:

1. Writing, modifying, or deleting user files or config
2. State-modifying shell or terminal commands (git commit, npm install, mkdir, and anything beyond read-only)
3. Outbound network requests using authentication or sending data externally
4. Sending emails, messages, or communications to any recipient
5. Reading, displaying, or transmitting credential values
6. Browser form submissions, interactive element clicks, or file downloads
7. Creating or modifying cron jobs, heartbeat config, or webhooks
8. Financial transactions (purchases, payments, billing changes)
9. Permission or access control changes
10. Any irreversible or bulk (5+ targets) operation

## Gate process

**IMPORTANT — Output efficiency:** Every message adds cost. Combine steps into a single response. Keep reasoning brief. Do not narrate steps separately. Read `{baseDir}/GUIDE.md` on first gated action for output format templates.

**Step 1 — Identify.** List the actions matching trigger criteria. State what, where, and what data is involved.

**Step 2 — Determine mode.** Check `GUARDRAILS_MODE` in config. If not set, auto-detect: direct user message → **full gate**; cron/heartbeat/webhook → **audit-only**. Apply `/guardrails strict` or `/guardrails relaxed` if active.

**Step 3 — Classify.** Read `{baseDir}/ACTIONS.md`. Assign base tier (2, 3, or 4).

**Step 4 — Escalate.** Read `{baseDir}/TIERS.md`. Apply escalation rules if applicable. Result is final tier.

**Step 5 — Pre-approval.** Steps 3 and 4 (classify and escalate) MUST complete before this check — pre-approval only skips the approval gate, never classification or escalation. If action matches `GUARDRAILS_PRE_APPROVED` AND final tier (after escalation) < 4, log with "PRE-APPROVED" tag and proceed. If escalation raised the action to Tier 4, pre-approval is void — apply full Tier 4 controls.

**Step 6 — Hard constraints.** Read `{baseDir}/POLICY.md`. If any NEVER rule is violated, BLOCK regardless of tier or approval.

**Step 7 — Controls.** Apply the universal control from `{baseDir}/TIERS.md` for the final tier × mode.

**Step 8 — Log and execute.** Append to `{baseDir}/guardrails_log.md`: timestamp, tier, type, description, mode, outcome, approval, trigger, escalation reason. Execute or halt.

**Note:** Writes to `guardrails_log.md` are exempt from the gate.

## Mode auto-detection

- **Full gate** — interactive session. Tier 3+ requires explicit approval.
- **Audit-only** — autonomous execution with logging and alerts. Tier 4 halted by default.
- **Strict** — all tiers +1. Via `/guardrails strict` or config.
- **Relaxed** — Tier 2 skips gate. Via `/guardrails relaxed` or config.

**Config overrides** (openclaw.json `skills.entries.runtime-guardrails.env`):
- `GUARDRAILS_MODE` — "auto", "full-gate", "audit-only", "strict"
- `GUARDRAILS_AUTONOMOUS_MODE` — "audit-only" (default), "full-gate"
- `GUARDRAILS_TIER4_AUTONOMOUS` — "halt" (default), "alert-and-execute"
- `GUARDRAILS_PRE_APPROVED` — comma-separated patterns (e.g. "write:briefings/*,send:telegram:self")
- `GUARDRAILS_ALERT_CHANNEL` — "auto" or specific channel

## Slash commands

| Command | Effect |
|---------|--------|
| `/guardrails status` | Session stats: tier distribution, approvals, denials, mode |
| `/guardrails log` | Last 20 log entries |
| `/guardrails policy` | Display hard constraints from POLICY.md |
| `/guardrails strict` | All tiers +1 for remainder of session |
| `/guardrails relaxed` | Tier 2 skips gate for remainder of session |

## Actions not in the catalog

1. Does it match trigger criteria? If not, proceed normally.
2. If it triggers the gate, classify by analogy to the most similar listed action.
3. If no analogy fits, default to Tier 3.
4. Log with "UNLISTED" tag for future catalog updates.
