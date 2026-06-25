# APEX Machine State Model
**Version:** 1.0  
**Purpose:** Defines the four operational modes of the build node, what each mode controls, how mode transitions work, and the rules the machine-control service enforces.

---

## The Four Modes

### Normal
**When to use:** Daytime, active development sessions, interactive use.

| Setting | Value |
|---|---|
| Fan profile | Balanced (Fan Control: "Balanced" profile) |
| RGB | On — full SignalRGB config |
| Display | On |
| CPU cooler LCD | On (clock/temp display) |
| Job throughput | Full — all job classes allowed |
| Overnight batch | Not triggered in this mode |
| Telegram notifications | All — including informational |

---

### Quiet Build
**When to use:** Background builds, evening hours, when noise matters but the machine is actively working.

| Setting | Value |
|---|---|
| Fan profile | Quiet (Fan Control: "Quiet" profile — reduced RPM targets) |
| RGB | On (no change — RGB doesn't make noise) |
| Display | On or sleep after 10 min inactivity |
| CPU cooler LCD | On |
| Job throughput | Moderate — Class 1 and 2 jobs run, heavy batch deferred |
| Overnight batch | No — use Covert Sleep for overnight |
| Telegram notifications | All |

---

### Covert Sleep
**When to use:** Overnight operation, when the machine needs to be visually and acoustically invisible.

| Setting | Value |
|---|---|
| Fan profile | Silent (Fan Control: "Silent" profile — minimum RPM, aggressive temp tolerance) |
| RGB | Off (all SignalRGB devices dark) |
| Display | Sleep immediately |
| CPU cooler LCD | Off (if controllable via iCUE — otherwise dim to minimum) |
| Job throughput | Low-priority only — Class 1 (Hermes/Ollama batch), no heavy inference |
| Overnight batch | Yes — this is the trigger for nightly jobs |
| Telegram notifications | Alerts and errors only — no informational noise |

**Jobs allowed in Covert Sleep:**
- Markdown regeneration
- Doc freshness checks
- Metric snapshots from ingestors
- Diff summarization
- Recommendation engine run
- Weekly review compilation (if scheduled)
- Context pack cleanup
- Database maintenance jobs

**Jobs NOT allowed in Covert Sleep:**
- Heavy Ollama/NIM inference (thermal + noise risk)
- GitHub repo operations requiring real-time response
- Dashboard-facing operations
- Any job classified `overnight_safe = false`

---

### High Performance
**When to use:** Heavy batch processing, large model inference, CPU-intensive builds.

| Setting | Value |
|---|---|
| Fan profile | Performance (Fan Control: "Performance" profile — maximum cooling) |
| RGB | On |
| Display | On |
| CPU cooler LCD | On |
| Job throughput | Maximum — all job classes, concurrent execution allowed |
| Overnight batch | Can be triggered manually |
| Telegram notifications | All |

---

## Mode Transition Rules

```
Normal      → Quiet Build     : Tier B (execute + notify)
Normal      → Covert Sleep    : Tier B (execute + notify)
Normal      → High Performance: Tier B (execute + notify)
Quiet Build → Normal          : Tier B
Quiet Build → Covert Sleep    : Tier B
Covert Sleep → Normal         : Tier B (wakes machine from silent mode)
Covert Sleep → Any other mode : Tier B
High Performance → Any        : Tier B

Any mode → Shutdown            : Tier D (confirm-to-execute, 60s window)
Any mode → Reboot              : Tier D (confirm-to-execute, 60s window)
```

Mode transitions triggered by UPS power events bypass the Tier B notification requirement — they execute automatically and log to `machine_state_history`.

---

## Telegram Commands

```
/mode normal           → Switch to Normal
/mode quiet            → Switch to Quiet Build
/mode covert           → Switch to Covert Sleep
/mode performance      → Switch to High Performance
/shutdown              → Initiate shutdown (Tier D)
/reboot                → Initiate reboot (Tier D)
```

---

## Hardware Control Implementation Notes

### Fan Control
- Fan Control (https://getfancontrol.com) is the primary fan management tool.
- APEX machine-control service switches Fan Control profiles by writing to its config file or using its CLI interface.
- Profile names must exactly match what is configured in Fan Control: "Balanced", "Quiet", "Silent", "Performance"
- If Fan Control is not running, log a warning and skip fan control — do not fail the mode switch.

### RGB (SignalRGB)
- SignalRGB is used for RGB control.
- APEX controls RGB by calling SignalRGB's CLI or REST API to toggle lighting off/on.
- RGB control is **best-effort** — if SignalRGB is not running or the API call fails, log and continue. RGB failure should never block a mode transition.
- On/off state: "off" means all devices set to solid black (0,0,0), not physically disabled.

### Display Sleep
- Controlled via OS display power commands.
- Linux: `xset dpms force off` or equivalent via systemd.
- On mode switch to Covert Sleep: `sleep 30 && xset dpms force off` (30-second grace period).
- On mode switch away from Covert Sleep: `xset dpms force on`.

### CPU Cooler LCD (Corsair iCUE)
- The Corsair iCUE Link CPU cooler has an LCD screen.
- If iCUE exposes a CLI or API for display control, use it to turn the LCD off in Covert Sleep.
- If not controllable programmatically, dim to minimum brightness via iCUE manually on first setup.
- Document the outcome in Phase 5 build notes — this is a nice-to-have, not a blocker.

---

## machine-control Service Interface

The `machine-control` service in `apex-control` exposes these internal endpoints for other APEX services to call:

```
GET  /machine/state              → current mode, fan profile, RGB state, display state
POST /machine/mode/{mode}        → request mode transition (triggers approval flow if Tier D)
POST /machine/shutdown           → initiate shutdown (triggers Tier D approval)
POST /machine/reboot             → initiate reboot (triggers Tier D approval)
GET  /machine/history            → recent mode transition history
POST /machine/jobs/pause         → pause all scheduled jobs (called by UPS safety service)
POST /machine/jobs/resume        → resume scheduled jobs
```

---

## Scheduled Mode Transitions (Optional)

The scheduler can trigger automatic mode transitions on a schedule. These are configured per-owner preference and off by default.

Example schedule (off by default, opt-in):
```
07:00 → switch to Normal
23:00 → switch to Covert Sleep
```

Scheduled transitions are Tier A (auto-execute) since they are pre-approved by being configured.
