# APEX Approval and Autonomy Policies
**Version:** 1.0  
**Purpose:** Defines every action category, its autonomy tier, and the confirmation requirements. This file governs what APEX can do on its own vs. what requires Hunter Lazenby's explicit approval via Telegram.

---

## The Four Autonomy Tiers

| Tier | Name | Description |
|---|---|---|
| **A** | Auto-execute | APEX executes immediately with no notification required |
| **B** | Execute + notify | APEX executes and sends a Telegram notification afterward |
| **C** | Draft + request approval | APEX drafts the action, sends it to Telegram, waits for /approve or /reject |
| **D** | Confirm-to-execute | APEX stages the action, requires explicit /confirm within 60 seconds, then executes |

**Default rule:** When in doubt, use Tier C. Never guess that something is safe enough for Tier A or B.

---

## Tier A — Auto-Execute (no notification required)

These are recurring, low-risk, fully reversible operations that would create noise if they required confirmation.

| Action | Conditions |
|---|---|
| Generate markdown handoff doc | After any build session ends |
| Run doc freshness check | Scheduled — checks commit timestamps vs doc timestamps |
| Snapshot metrics from ingestors | Scheduled — GA4, Plausible, Supabase, etc. |
| Rotate recommendation status to 'reviewed' | After 7 days in 'new' state with no response |
| Add note from Telegram `/note` command | Owner-sent only (chat ID verified) |
| Add task from Telegram `/task` command | Owner-sent only, priority = medium or lower |
| Send daily summary report | Scheduled morning job |
| Save context pack | Part of any model handoff |
| Log model run | After every AI execution |
| Regenerate APEX.md for a project | After any doc section update |
| Update doc_sync_states | After every sync check |
| Run diff summarizer | After GitHub commit detected |
| Archive completed recommendations | When status = 'done' |

---

## Tier B — Execute + Notify (Telegram notification sent after)

These are safe but meaningful enough that the owner should know they happened.

| Action | Notification Content |
|---|---|
| Switch machine mode (Normal ↔ Quiet Build) | "Machine mode changed to [mode] at [time]" |
| Deploy Hermes overnight batch job start | "Overnight batch started. Jobs: [list]. Est. completion: [time]" |
| Mark a known issue as resolved | "Issue KI-[NNN] marked resolved on [project]" |
| Add a critical recommendation | "New critical recommendation for [project]: [title]" |
| Project completeness score drops below 80 | "⚠️ [project] completeness score dropped to [X]. Missing: [sections]" |
| Idea scoring complete | "Idea scored: [title] — [score]/100 — Recommendation: [action]" |
| Ingestor connection failure (auto-retry) | "⚠️ [service] ingestor failed. Retrying. Last success: [time]" |
| UPS goes off battery (power restored) | "✅ Power restored. UPS back on mains. Runtime: [X]min remaining." |

---

## Tier C — Draft + Request Approval (Telegram approval required)

These actions have meaningful consequences and need owner eyes before execution. APEX drafts the action clearly and sends it to Telegram. Owner replies `/approve [id]` or `/reject [id]`.

**No expiry on Tier C approvals** — they wait until responded to or manually cancelled.

| Action | What APEX sends | What approval triggers |
|---|---|---|
| Create a new GitHub repo | Repo name, description, visibility, initial files | Repo creation via GitHub API |
| Promote an idea to a project | Full idea summary, score breakdown, recommended type | Creates project record, scaffolds docs |
| Add a high-priority task (critical) | Task title, description, estimated effort | Task created in DB |
| Mark a project as 'live' | Project name, checklist of live-readiness criteria met | Stage updated to 'live' |
| Mark a project as 'paused' or 'retired' | Project name, reason | Stage updated |
| Deploy a scheduled job change | Job name, new schedule, affected projects | Job updated in APScheduler |
| Add a new cost item > $10/mo | Service name, cost, justification | Cost item added to DB |
| Trigger a scaffold generation | Project name, scaffold plan preview | Files created in repo |
| Promote idea to 'build-now' status | Full scored idea | Status updated, queued for Phase 8 flow |
| Send a bulk Telegram message | Message preview | Message sent |
| Change model routing rules | Old rule vs new rule diff | Routing config updated |
| Flag a project for kill-score review | Project name, kill-score signals | Kill score recommendation created |
| Trigger weekly review compilation | Scope of review | Review generated and sent |

---

## Tier D — Confirm-to-Execute (60-second window, high-risk)

These actions are irreversible or have financial, security, or operational consequences that demand deliberate confirmation. APEX stages the action, sends a Telegram message with a countdown, waits for `/confirm`. If no response in 60 seconds, the action is **automatically cancelled** and logged.

The `/confirm` message must come from the owner chat ID. Any other sender is ignored.

| Action | Risk | What happens on confirm |
|---|---|---|
| Shutdown build node | Operational | Graceful container stop → OS shutdown |
| Reboot build node | Operational | Graceful restart |
| Delete any project record | Data loss | Soft delete (deleted_at set) — not hard delete |
| Delete any task, decision, or handoff | Data loss | Soft delete |
| Deploy to a live project (GearGameDaily, DIY Cut List) | Production | Git-triggered or manual deploy executed |
| Write or rotate a secret/API key | Security | Secret updated in .env or hosting platform env vars |
| Any Stripe billing change | Financial | Pricing or plan config updated |
| BATCH: any live strategy parameter change | Financial + operational | Strategy config updated in live environment |
| BATCH: promote challenger to champion | Financial + operational | Champion designation updated, old champion archived |
| BATCH: enable live trading after pause | Financial | Trading re-enabled |
| BATCH: increase position size limit | Financial | Risk limit updated in code config |
| Coinbase API credential rotation | Security + financial | New credentials applied |
| Add or remove Cloudflare Access policy | Security | Access policy updated |
| Change Telegram owner chat ID | Security | Auth filter updated — ⚠️ Locks out current owner if wrong |

---

## Special Case: UPS Power Emergency

When UPS battery runtime drops below the threshold (default: 5 minutes remaining), APEX initiates an **automatic safe shutdown sequence** without waiting for approval. This is the one exception to Tier D confirmation — waiting 60 seconds for a confirm when the machine has 5 minutes of battery is not safe.

```
UPS Event Flow:
1. Power loss detected → Tier B notification sent immediately
2. Runtime drops below 15 min → Tier C approval request sent ("Shutdown in ~15 min unless power restores")
3. Runtime drops below 5 min → AUTO EXECUTE: pause queue → save state → stop containers → shutdown
4. If power restores at any point → cancel pending shutdown, send Tier B notification
```

The 15-minute threshold (Tier C) and 5-minute threshold (auto-execute) are configurable in APEX settings.

---

## BATCH-Specific Override Policy

BATCH is a governed system and has stricter rules that override the general tiers above:

| Situation | Default Tier | BATCH Override |
|---|---|---|
| Any live configuration change | C | D (mandatory) |
| Any real-money parameter change | C | D (mandatory) |
| Challenger promotion | C | D (mandatory) |
| Enable live trading | C | D with explicit capital confirmation |
| Emergency stop (/stop_all) | D | A — executes immediately, notifies after |

The BATCH emergency stop (`/stop_all`) is the only Tier D action that is demoted to Tier A. Speed matters more than confirmation when stopping a live trading system.

---

## Telegram Confirmation Format

When APEX sends a Tier C or D approval request, the message format is:

```
⚠️ [TIER C / TIER D] Approval Request
━━━━━━━━━━━━━━━━━━━━━━━
Action: [action title]
Project: [project name or APEX-SYSTEM]
Risk: [low/medium/high/critical]
━━━━━━━━━━━━━━━━━━━━━━━
[Plain-English description of exactly what will happen]
━━━━━━━━━━━━━━━━━━━━━━━
ID: APR-[UUID-SHORT]
[TIER D only]: ⏱ Auto-cancels in 60 seconds if no response.

Reply: /approve APR-[ID] or /reject APR-[ID]
```

---

## Audit Log

Every approval request — regardless of outcome — is written to the `approval_requests` table and never deleted. This provides a full audit trail of every consequential action APEX took or attempted to take.

Fields logged: action type, risk level, what was proposed, who requested it, timestamp, response (approved/rejected/expired), who responded, and when.
