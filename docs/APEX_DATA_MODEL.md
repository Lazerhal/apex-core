# APEX Data Model
**Version:** 1.0  
**Purpose:** Authoritative definition of every entity, field, relationship, and constraint in the APEX PostgreSQL database. This file is the spec that the backend team (Claude Code) implements against. No table is created without a definition here first.

---

## Design Principles

- **UUIDs** for all primary keys — no integer sequences that leak row counts
- **created_at / updated_at** on every table — automatic via PostgreSQL trigger
- **Soft deletes** on Project, Idea — `deleted_at` nullable timestamp, never hard delete
- **JSONB** for flexible extension fields — typed extensions live in their own tables, but free-form notes and metadata go in JSONB to avoid schema churn
- **Enums as PostgreSQL enums** — not varchar — enforced at DB level

---

## Core Entities

---

### `projects`

The central table. Every managed project has exactly one row here.

```sql
CREATE TABLE projects (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    apex_id         VARCHAR(30) UNIQUE NOT NULL,  -- e.g. APEX-SAAS-202606-001
    name            VARCHAR(60) NOT NULL,
    type            project_type NOT NULL,
    stage           project_stage NOT NULL,
    owner           VARCHAR(100) NOT NULL DEFAULT 'Hunter Lazenby',
    short_description TEXT,
    goal            TEXT,
    status_note     TEXT,
    live_url        VARCHAR(500),
    internal_url    VARCHAR(500),
    github_repo     VARCHAR(500),
    repo_visibility VARCHAR(10) CHECK (repo_visibility IN ('public','private')),
    primary_branch  VARCHAR(50) DEFAULT 'main',
    local_path      VARCHAR(500),
    monthly_cost_usd NUMERIC(10,2) DEFAULT 0,
    monthly_revenue_usd NUMERIC(10,2) DEFAULT 0,
    cost_efficiency_flag VARCHAR(20) DEFAULT 'healthy'
        CHECK (cost_efficiency_flag IN ('healthy','watch','inefficient')),
    completeness_score INTEGER DEFAULT 0 CHECK (completeness_score BETWEEN 0 AND 100),
    metadata        JSONB DEFAULT '{}',
    deleted_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TYPE project_type AS ENUM (
    'product-tool', 'content-engine', 'saas', 'governed-system', 'infrastructure', 'idea'
);

CREATE TYPE project_stage AS ENUM (
    'idea', 'planning', 'active-development', 'live', 'paused', 'retired'
);
```

---

### `ideas`

Ideas that have not yet become projects. Scored and tracked here before promotion.

```sql
CREATE TABLE ideas (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title           VARCHAR(120) NOT NULL,
    raw_description TEXT NOT NULL,          -- free-form as entered
    structured_description TEXT,            -- after clarification agent processes it
    clarification_state JSONB DEFAULT '{}', -- Q&A history from clarification agent
    score_total     INTEGER,                -- 0-100
    score_breakdown JSONB,                  -- {factor: score, ...}
    score_explanation TEXT,
    recommendation  VARCHAR(30)             -- 'build-now' | 'backlog' | 'research-more' | 'reject'
        CHECK (recommendation IN ('build-now','backlog','research-more','reject')),
    promoted_to_project_id UUID REFERENCES projects(id),
    status          VARCHAR(20) DEFAULT 'new'
        CHECK (status IN ('new','clarifying','scored','accepted','rejected','promoted')),
    source          VARCHAR(50) DEFAULT 'telegram',  -- where the idea came from
    deleted_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

### `tasks`

Tasks for any project. Hierarchical via parent_task_id.

```sql
CREATE TABLE tasks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    parent_task_id  UUID REFERENCES tasks(id),  -- for subtasks
    apex_task_id    VARCHAR(10),                -- e.g. T-001 (scoped to project)
    title           VARCHAR(300) NOT NULL,
    description     TEXT,
    priority        task_priority NOT NULL DEFAULT 'medium',
    type            task_type NOT NULL DEFAULT 'feature',
    estimated_effort VARCHAR(20)
        CHECK (estimated_effort IN ('trivial','small','medium','large','xl')),
    assigned_to     VARCHAR(100),               -- model or tool name
    blocked_by      TEXT[],                     -- array of task IDs or descriptions
    status          VARCHAR(20) DEFAULT 'open'
        CHECK (status IN ('open','in-progress','blocked','done','cancelled')),
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TYPE task_priority AS ENUM ('critical','high','medium','low','backlog');
CREATE TYPE task_type AS ENUM ('bug','feature','improvement','infrastructure','documentation','research','security');
```

---

### `notes`

Free-form notes attached to any project. Quick capture from Telegram or dashboard.

```sql
CREATE TABLE notes (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id  UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    content     TEXT NOT NULL,
    source      VARCHAR(50) DEFAULT 'dashboard',  -- 'telegram' | 'dashboard' | 'claude-code' | 'manual'
    tags        TEXT[] DEFAULT '{}',
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

### `decisions`

Structured decision log per project. Immutable once created — append-only.

```sql
CREATE TABLE decisions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    apex_decision_id VARCHAR(10),             -- e.g. D-001
    date            DATE NOT NULL DEFAULT CURRENT_DATE,
    title           VARCHAR(200) NOT NULL,
    context         TEXT NOT NULL,
    options_considered TEXT[],
    decision_made   TEXT NOT NULL,
    rationale       TEXT NOT NULL,
    made_by         VARCHAR(100) DEFAULT 'Hunter Lazenby',
    revisit_trigger TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
    -- intentionally no updated_at: decisions are append-only
);
```

---

### `known_issues`

Known bugs, security issues, and tech debt. Separate from tasks — these are observed problems, not planned work items.

```sql
CREATE TABLE known_issues (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    apex_issue_id   VARCHAR(10),
    description     TEXT NOT NULL,
    severity        issue_severity NOT NULL,
    type            issue_type NOT NULL,
    discovered_date DATE,
    workaround      TEXT,
    fix_status      VARCHAR(20) DEFAULT 'open'
        CHECK (fix_status IN ('open','in-progress','wont-fix','resolved')),
    resolved_at     TIMESTAMPTZ,
    linked_task_id  UUID REFERENCES tasks(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TYPE issue_severity AS ENUM ('critical','high','medium','low');
CREATE TYPE issue_type AS ENUM ('bug','security','performance','ux','debt','missing-feature');
```

---

### `risks`

Strategic and operational risks per project.

```sql
CREATE TABLE risks (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id  UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    apex_risk_id VARCHAR(10),
    title       VARCHAR(200) NOT NULL,
    description TEXT NOT NULL,
    likelihood  VARCHAR(10) NOT NULL CHECK (likelihood IN ('low','medium','high')),
    impact      VARCHAR(10) NOT NULL CHECK (impact IN ('low','medium','high','critical')),
    mitigation  TEXT NOT NULL,
    status      VARCHAR(20) DEFAULT 'open'
        CHECK (status IN ('open','mitigated','accepted','resolved')),
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

### `cost_items`

Line-item costs per project. Summed to populate projects.monthly_cost_usd.

```sql
CREATE TABLE cost_items (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id  UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name        VARCHAR(100) NOT NULL,
    type        VARCHAR(20) NOT NULL
        CHECK (type IN ('fixed-monthly','fixed-annual','per-use','free')),
    amount_usd  NUMERIC(10,4) NOT NULL DEFAULT 0,
    billing_date VARCHAR(50),               -- e.g. "1st of month" or "annually in March"
    notes       TEXT,
    active      BOOLEAN DEFAULT TRUE,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

### `services`

External services and integrations a project depends on.

```sql
CREATE TABLE services (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name            VARCHAR(100) NOT NULL,
    purpose         TEXT NOT NULL,
    tier            VARCHAR(50),
    monthly_cost_usd NUMERIC(10,2) DEFAULT 0,
    dashboard_url   VARCHAR(500),
    criticality     VARCHAR(20) NOT NULL DEFAULT 'important'
        CHECK (criticality IN ('critical','important','optional')),
    status          VARCHAR(20) DEFAULT 'active'
        CHECK (status IN ('active','degraded','down','deprecated')),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

### `env_var_requirements`

Documents required environment variables per project. Names only — never values.

```sql
CREATE TABLE env_var_requirements (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id  UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    var_name    VARCHAR(100) NOT NULL,
    purpose     TEXT NOT NULL,
    required    BOOLEAN DEFAULT TRUE,
    stored_in   VARCHAR(200),              -- e.g. "Railway env vars" or "local .env"
    sensitive   BOOLEAN DEFAULT TRUE,
    present     BOOLEAN DEFAULT FALSE,     -- updated by APEX when key is confirmed present
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

### `metric_snapshots`

Point-in-time metric readings per project. Collected by ingestors on schedule.

```sql
CREATE TABLE metric_snapshots (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id  UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    metric_name VARCHAR(100) NOT NULL,      -- e.g. 'monthly_sessions', 'mrr_usd', 'error_rate'
    value       NUMERIC(14,4),
    value_text  VARCHAR(200),              -- for non-numeric values
    unit        VARCHAR(30),               -- e.g. 'USD', 'sessions', 'percent'
    source      VARCHAR(50) NOT NULL,      -- e.g. 'plausible', 'stripe', 'supabase', 'manual'
    period_start DATE,
    period_end  DATE,
    snapshot_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Indexed for fast per-project time-series queries
CREATE INDEX idx_metric_snapshots_project_metric
    ON metric_snapshots(project_id, metric_name, snapshot_at DESC);
```

---

### `recommendations`

Generated by the APEX recommendation engine. One row per recommendation.

```sql
CREATE TABLE recommendations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    generated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    type            VARCHAR(30) NOT NULL
        CHECK (type IN ('bug-fix','ux-improvement','performance','cost-reduction',
                        'traffic-opportunity','monetisation','reuse-suggestion',
                        'doc-fix','experiment','kill-score','security')),
    severity        VARCHAR(10) NOT NULL CHECK (severity IN ('critical','high','medium','low')),
    title           VARCHAR(200) NOT NULL,
    rationale       TEXT NOT NULL,
    supporting_signals TEXT[],
    estimated_effort VARCHAR(10) CHECK (estimated_effort IN ('trivial','small','medium','large','xl')),
    expected_benefit TEXT,
    suggested_action TEXT,
    status          VARCHAR(20) DEFAULT 'new'
        CHECK (status IN ('new','reviewed','accepted','rejected','in-progress','done')),
    reviewed_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

### `handoffs`

One per build session. Records what was done, what remains, and instructions for next session.

```sql
CREATE TABLE handoffs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    session_date    DATE NOT NULL DEFAULT CURRENT_DATE,
    session_tool    VARCHAR(100),           -- e.g. 'Claude Code', 'Grok Build', 'manual'
    what_was_done   TEXT NOT NULL,
    what_remains    TEXT NOT NULL,
    what_is_blocked TEXT NOT NULL DEFAULT 'Nothing',
    next_model_should TEXT NOT NULL,
    context_pack_id VARCHAR(100),           -- reference to stored context pack
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
    -- append-only: no updated_at
);
```

---

### `model_runs`

Every AI model execution is logged here. Powers the credit watchdog.

```sql
CREATE TABLE model_runs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID REFERENCES projects(id),  -- nullable: some runs are APEX-internal
    task_type       VARCHAR(100) NOT NULL,
    model           VARCHAR(100) NOT NULL,          -- e.g. 'claude-sonnet-4-6'
    provider        VARCHAR(50) NOT NULL,           -- e.g. 'anthropic', 'deepseek', 'ollama'
    routing_class   VARCHAR(10) NOT NULL,           -- '1', '1.5', '2', '2a', '3', '4', '5', '6'
    prompt_package_id VARCHAR(100),
    input_tokens    INTEGER,
    output_tokens   INTEGER,
    cost_usd        NUMERIC(10,6),
    data_sensitivity VARCHAR(10) DEFAULT 'low'
        CHECK (data_sensitivity IN ('low','medium','high')),
    quality_score   INTEGER CHECK (quality_score BETWEEN 1 AND 5),
    fallback_occurred BOOLEAN DEFAULT FALSE,
    fallback_reason VARCHAR(200),
    duration_ms     INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- For credit watchdog queries
CREATE INDEX idx_model_runs_project_provider_date
    ON model_runs(project_id, provider, created_at DESC);
```

---

### `reusable_assets`

Tracks code modules, prompt templates, UI patterns, and other reusable artifacts across projects.

```sql
CREATE TABLE reusable_assets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(100) NOT NULL,
    type            VARCHAR(30) NOT NULL
        CHECK (type IN ('code-module','prompt-template','integration-adapter',
                        'ui-pattern','auth-block','billing-block',
                        'analytics-pattern','doc-template','deployment-script')),
    description     TEXT NOT NULL,
    source_project_id UUID REFERENCES projects(id),
    repo_path       VARCHAR(500),           -- where the asset lives
    used_by_projects UUID[],               -- project IDs that reference this asset
    tags            TEXT[] DEFAULT '{}',
    quality_score   INTEGER CHECK (quality_score BETWEEN 1 AND 5),
    last_validated  DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

### `approval_requests`

Every action requiring owner confirmation goes through here before execution.

```sql
CREATE TABLE approval_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID REFERENCES projects(id),
    type            VARCHAR(50) NOT NULL,   -- see approval_policies.md for full list
    risk_level      VARCHAR(10) NOT NULL CHECK (risk_level IN ('low','medium','high','critical')),
    title           VARCHAR(200) NOT NULL,
    description     TEXT NOT NULL,
    proposed_action JSONB NOT NULL,         -- machine-readable action payload
    requested_by    VARCHAR(100) NOT NULL,  -- 'apex-system' | 'hermes-agent' | model name
    status          VARCHAR(20) DEFAULT 'pending'
        CHECK (status IN ('pending','approved','rejected','expired','cancelled')),
    expires_at      TIMESTAMPTZ,            -- NULL = no expiry; 60s for high-risk commands
    telegram_message_id VARCHAR(100),       -- for editing the Telegram approval message
    responded_at    TIMESTAMPTZ,
    response_note   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Only one pending approval of same type per project at a time
CREATE UNIQUE INDEX idx_approval_requests_pending_unique
    ON approval_requests(project_id, type)
    WHERE status = 'pending';
```

---

### `machine_state`

Single-row table (plus history). Tracks build node operational state.

```sql
CREATE TABLE machine_state (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    mode        VARCHAR(20) NOT NULL DEFAULT 'normal'
        CHECK (mode IN ('normal','quiet-build','covert-sleep','high-performance')),
    fan_profile VARCHAR(50),
    rgb_state   VARCHAR(20) DEFAULT 'on' CHECK (rgb_state IN ('on','off','minimal')),
    display_state VARCHAR(20) DEFAULT 'on' CHECK (display_state IN ('on','off','sleep')),
    jobs_paused BOOLEAN DEFAULT FALSE,
    ups_on_battery BOOLEAN DEFAULT FALSE,
    ups_battery_pct INTEGER,
    ups_runtime_minutes INTEGER,
    notes       TEXT,
    set_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    set_by      VARCHAR(50) DEFAULT 'manual'  -- 'manual' | 'telegram' | 'schedule' | 'ups-event'
);

CREATE TABLE machine_state_history (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    previous_mode   VARCHAR(20),
    new_mode        VARCHAR(20),
    changed_by      VARCHAR(50),
    reason          TEXT,
    changed_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

### `power_events`

UPS and power event history.

```sql
CREATE TABLE power_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_type      VARCHAR(30) NOT NULL
        CHECK (event_type IN ('power-loss','power-restored','low-battery',
                              'critical-battery','graceful-shutdown','forced-shutdown')),
    ups_battery_pct_at_event INTEGER,
    ups_runtime_minutes_at_event INTEGER,
    action_taken    TEXT,                   -- what APEX did in response
    telegram_notified BOOLEAN DEFAULT FALSE,
    jobs_paused     BOOLEAN DEFAULT FALSE,
    shutdown_initiated BOOLEAN DEFAULT FALSE,
    resolved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

### `doc_sync_states`

Per-project markdown sync status. Updated by the doc engine on every sync check.

```sql
CREATE TABLE doc_sync_states (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id          UUID NOT NULL UNIQUE REFERENCES projects(id) ON DELETE CASCADE,
    sync_status         VARCHAR(20) NOT NULL DEFAULT 'in_sync'
        CHECK (sync_status IN ('in_sync','markdown_ahead','db_ahead','conflict')),
    last_db_write       TIMESTAMPTZ,
    last_markdown_write TIMESTAMPTZ,
    last_sync_check     TIMESTAMPTZ,
    last_github_commit  TIMESTAMPTZ,
    freshness_warning   BOOLEAN DEFAULT FALSE,  -- true when commit newer than last doc update
    conflict_details    TEXT,
    schema_version      VARCHAR(10) DEFAULT '1.0',
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

### `scheduled_jobs`

APScheduler job registry — backed by SQLAlchemy so jobs persist across restarts.
*Note: APScheduler manages this table automatically via SQLAlchemyJobStore. Documented here for reference.*

```sql
-- Managed by APScheduler SQLAlchemyJobStore
-- Table name: apscheduler_jobs (auto-created by APScheduler)
-- Key fields: id, next_run_time, job_state (pickled)
-- We wrap this with our own jobs_registry for human-readable job tracking:

CREATE TABLE jobs_registry (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    apscheduler_id  VARCHAR(100) UNIQUE,    -- matches APScheduler job id
    name            VARCHAR(100) NOT NULL,
    description     TEXT,
    schedule        VARCHAR(100),           -- human-readable e.g. "daily at 02:00"
    last_run        TIMESTAMPTZ,
    last_result     VARCHAR(20),            -- 'success' | 'error' | 'skipped'
    last_error      TEXT,
    run_in_mode     TEXT[],                 -- which machine modes allow this job
    overnight_safe  BOOLEAN DEFAULT FALSE,  -- safe for covert-sleep mode
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

### `experiments`

A/B tests and experiments tracked per project.

```sql
CREATE TABLE experiments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    apex_exp_id     VARCHAR(10),
    title           VARCHAR(200) NOT NULL,
    hypothesis      TEXT NOT NULL,
    method          TEXT NOT NULL,
    start_date      DATE NOT NULL,
    end_date        DATE,
    result          TEXT,
    conclusion      VARCHAR(20)
        CHECK (conclusion IN ('winner','loser','inconclusive','in-progress')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## Project-Type Extension Tables

These tables extend the core `projects` table with type-specific fields.

---

### `project_ext_product_tool`

```sql
CREATE TABLE project_ext_product_tool (
    project_id          UUID PRIMARY KEY REFERENCES projects(id) ON DELETE CASCADE,
    maturity_level      VARCHAR(20) DEFAULT 'prototype'
        CHECK (maturity_level IN ('prototype','v1-mvp','polished','mature')),
    monetisation_options TEXT[],
    distribution_channels TEXT[],
    premium_upgrade_roadmap TEXT,
    marketplace_listings JSONB DEFAULT '[]'  -- [{platform, url, price_usd, active}]
);
```

---

### `project_ext_content_engine`

```sql
CREATE TABLE project_ext_content_engine (
    project_id          UUID PRIMARY KEY REFERENCES projects(id) ON DELETE CASCADE,
    publishing_cadence  VARCHAR(100),
    content_pipeline    TEXT,
    content_runway      VARCHAR(200),
    seo_status          TEXT,
    analytics_status    TEXT,
    scheduler_health    VARCHAR(20) DEFAULT 'unknown'
        CHECK (scheduler_health IN ('healthy','degraded','down','unknown')),
    monetisation_sources JSONB DEFAULT '[]',  -- [{source, status, monthly_revenue_est}]
    promotion_channels  TEXT[],
    image_sources       TEXT[]
);
```

---

### `project_ext_saas`

```sql
CREATE TABLE project_ext_saas (
    project_id              UUID PRIMARY KEY REFERENCES projects(id) ON DELETE CASCADE,
    auth_flow               TEXT,
    billing_flow            TEXT,
    pricing_model           JSONB DEFAULT '[]',  -- [{name, price_usd, billing_period, features}]
    free_tier_limits        TEXT,
    free_tier_enforcement   VARCHAR(20) DEFAULT 'client-side'
        CHECK (free_tier_enforcement IN ('server-side','client-side','mixed')),
    conversion_funnel       TEXT,
    churn_risks             TEXT[],
    growth_levers           TEXT[],
    affiliate_setup         JSONB DEFAULT '[]',  -- [{network, tag, status, integration_method}]
    mrr_usd                 NUMERIC(10,2) DEFAULT 0,
    active_subscribers      INTEGER DEFAULT 0
);
```

---

### `project_ext_governed_system`

```sql
CREATE TABLE project_ext_governed_system (
    project_id              UUID PRIMARY KEY REFERENCES projects(id) ON DELETE CASCADE,
    current_stage           VARCHAR(30) DEFAULT 'stage-0-architecture'
        CHECK (current_stage IN (
            'stage-0-architecture','stage-1-strategy',
            'stage-2-paper','stage-3-small-live','stage-4-expansion'
        )),
    stage_gate_status       JSONB DEFAULT '{}',
    governance_policy       TEXT,
    approval_tree           TEXT,
    risk_limits             JSONB DEFAULT '[]',
    champion_strategy       TEXT,
    challenger_strategies   TEXT[],
    promotion_criteria      TEXT,
    emergency_controls      TEXT[],
    safety_constraints      TEXT[],
    reference_frameworks    TEXT[]
);
```

---

## Relationships Summary

```
projects
  ├── tasks (1:many)
  │     └── tasks (self-ref: subtasks)
  ├── notes (1:many)
  ├── decisions (1:many, append-only)
  ├── known_issues (1:many)
  │     └── tasks (linked_task_id FK)
  ├── risks (1:many)
  ├── cost_items (1:many)
  ├── services (1:many)
  ├── env_var_requirements (1:many)
  ├── metric_snapshots (1:many, time-series)
  ├── recommendations (1:many)
  ├── handoffs (1:many, append-only)
  ├── experiments (1:many)
  ├── doc_sync_states (1:1)
  ├── project_ext_product_tool (1:0..1)
  ├── project_ext_content_engine (1:0..1)
  ├── project_ext_saas (1:0..1)
  └── project_ext_governed_system (1:0..1)

ideas
  └── projects (promoted_to_project_id FK, nullable)

model_runs
  └── projects (project_id FK, nullable — some runs are APEX-internal)

reusable_assets
  └── projects (source_project_id FK, nullable)

approval_requests
  └── projects (project_id FK, nullable — some approvals are system-level)

machine_state (single-row, current state)
machine_state_history (append-only log)
power_events (append-only log)
jobs_registry (mirrors APScheduler)
```

---

## Indexes (beyond primary keys and FKs)

```sql
-- Fast project lookups
CREATE INDEX idx_projects_type ON projects(type);
CREATE INDEX idx_projects_stage ON projects(stage);
CREATE INDEX idx_projects_apex_id ON projects(apex_id);

-- Task queries
CREATE INDEX idx_tasks_project_status ON tasks(project_id, status);
CREATE INDEX idx_tasks_priority ON tasks(priority);

-- Recommendation engine queries
CREATE INDEX idx_recommendations_project_status ON recommendations(project_id, status);
CREATE INDEX idx_recommendations_severity ON recommendations(severity);

-- Handoff: latest per project
CREATE INDEX idx_handoffs_project_date ON handoffs(project_id, session_date DESC);

-- Model run cost tracking
CREATE INDEX idx_model_runs_cost ON model_runs(provider, created_at DESC);
CREATE INDEX idx_model_runs_project ON model_runs(project_id, created_at DESC);

-- Ideas pipeline
CREATE INDEX idx_ideas_status ON ideas(status);
CREATE INDEX idx_ideas_score ON ideas(score_total DESC NULLS LAST);
```

---

## Migration Strategy

All schema changes go through versioned migration files using **Alembic** (FastAPI/SQLAlchemy standard). No raw SQL executed against production without a migration file.

```
migrations/
├── env.py
├── script.py.mako
└── versions/
    ├── 0001_initial_schema.py
    ├── 0002_add_metric_snapshots_index.py
    └── ...
```

Migration naming: `{NNNN}_{short_description}.py`  
Never edit a migration that has been applied to production.
