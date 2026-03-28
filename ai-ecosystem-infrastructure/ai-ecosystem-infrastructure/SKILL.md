---
name: ai-ecosystem-infrastructure
description: >
  Master skill for orchestrating the end-to-end AI Ecosystem transformation project — spanning Asset Governance,
  Attendance, Performance/Scorecard, MaxBill, Finance, and Operations modules. Trigger this skill whenever the user
  mentions any of the following: building or migrating infrastructure, AI ecosystem planning, module development
  (asset, attendance, HR, billing, finance, ops), system architecture decisions, data pipeline design, API
  integration specs, database schema design, workflow automation, dashboard or reporting logic, or any task
  related to transitioning from manual processes to AI-driven automation. Also trigger when the user asks Manus
  to generate code scaffolding, write specs, plan sprints, review module logic, or connect modules together
  within this ecosystem.
compatibility:
  tools:
    - code_execution
    - file_manager
    - web_search
    - terminal
  languages:
    - Python
    - TypeScript / JavaScript
    - SQL
    - YAML / JSON
---

# AI Ecosystem Infrastructure Skill

## Overview

This skill guides Manus through building a **gigantic AI Ecosystem** — a unified platform that replaces manual,
siloed operations with intelligent, interconnected modules. The system covers:

| Module | Core Function |
|---|---|
| **Asset Governance** | Track, audit, and lifecycle-manage all physical and digital assets |
| **Attendance** | Automate check-in/out, leave, shift scheduling, and anomaly detection |
| **Performance / Scorecard** | KPI tracking, appraisal workflows, AI-driven scoring |
| **MaxBill** | Billing, invoicing, collections, and revenue recognition |
| **Finance** | Budgeting, GL, AP/AR, reporting, forecasting |
| **Operations (Ops)** | Workflow orchestration, SLA tracking, resource allocation |

All modules share a **common data layer**, a **unified auth/RBAC system**, and expose **REST + event-driven APIs**.

---

## Guiding Principles

1. **Modularity first** — each domain is an independently deployable service that communicates via well-defined contracts.
2. **AI-augmented, not AI-replaced** — surface AI recommendations and automation alongside human review workflows.
3. **Single source of truth** — a shared entity registry (Org, Employee, Asset, Cost Centre) eliminates duplication.
4. **Audit by default** — every write operation produces an immutable audit trail.
5. **Progressive migration** — manual processes run in parallel until each AI module reaches >95% accuracy threshold.

---

## System Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    API Gateway / BFF                    │
│         (Auth, Rate Limit, Routing, Versioning)         │
└────────┬───────────────────────────────────┬────────────┘
         │ REST / GraphQL                    │ Events (Kafka / NATS)
┌────────▼────────┐                 ┌────────▼────────────┐
│  Domain Services│                 │   Event Bus          │
│  ─────────────  │                 │  ─────────────────── │
│  asset-svc      │◄───────────────►│  asset.events        │
│  attendance-svc │◄───────────────►│  attendance.events   │
│  performance-svc│◄───────────────►│  performance.events  │
│  billing-svc    │◄───────────────►│  billing.events      │
│  finance-svc    │◄───────────────►│  finance.events      │
│  ops-svc        │◄───────────────►│  ops.events          │
└────────┬────────┘                 └─────────────────────┘
         │
┌────────▼────────────────────────────────────────────────┐
│              Shared Data Layer                          │
│  Postgres (transactional) │ Redis (cache/session)       │
│  S3 / MinIO (documents)   │ ClickHouse (analytics)      │
└─────────────────────────────────────────────────────────┘
         │
┌────────▼────────────────────────────────────────────────┐
│              AI / ML Layer                              │
│  Model inference │ Anomaly detection │ Forecasting      │
│  Embedding store │ Audit explainability engine          │
└─────────────────────────────────────────────────────────┘
```

---

## Module Specifications

### 1. Asset Governance (`asset-svc`)

**Purpose:** Full lifecycle management of physical and digital assets — from procurement to disposal.

**Key Entities:** `Asset`, `AssetCategory`, `AssetAssignment`, `MaintenanceRecord`, `DepreciationSchedule`

**Core Workflows:**
- Asset registration with QR/barcode generation
- Assignment to employee / department / location
- Scheduled and ad-hoc maintenance tracking
- Straight-line and reducing-balance depreciation engine
- AI-powered: predict maintenance due dates, flag anomalous asset movements

**Data Schema (core):**
```sql
assets (
  id UUID PRIMARY KEY,
  code TEXT UNIQUE NOT NULL,         -- auto-generated barcode/QR ref
  name TEXT NOT NULL,
  category_id UUID REFERENCES asset_categories,
  status TEXT CHECK (status IN ('active','under_maintenance','retired','disposed')),
  purchase_date DATE,
  purchase_value NUMERIC(18,2),
  current_value NUMERIC(18,2),       -- updated by depreciation engine
  location_id UUID,
  assigned_to UUID REFERENCES employees,
  last_audit_at TIMESTAMPTZ,
  metadata JSONB,                    -- flexible attrs per category
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
)
```

**AI Hook:** On every assignment change or maintenance event, emit `asset.updated` → AI layer re-scores asset health.

---

### 2. Attendance (`attendance-svc`)

**Purpose:** Capture, validate, and analyse employee time — replacing manual registers with AI-verified records.

**Key Entities:** `AttendanceRecord`, `ShiftSchedule`, `LeaveRequest`, `Holiday`, `OvertimeEntry`

**Core Workflows:**
- Check-in / check-out via biometric, mobile GPS, or QR kiosk
- Shift scheduling with conflict detection
- Leave request → approval → balance deduction pipeline
- Late/early/absent auto-classification
- AI-powered: detect buddy-punching patterns, predict absenteeism risk

**Calculation Rules (implement exactly):**
```
effective_hours = check_out - check_in - break_minutes
overtime = MAX(0, effective_hours - shift_standard_hours)
late_minutes = MAX(0, check_in - shift_start)
status = CASE
  WHEN no record AND not on leave → 'absent'
  WHEN late_minutes > grace_period → 'late'
  WHEN effective_hours < 0.5 * shift_standard_hours → 'half_day'
  ELSE 'present'
END
```

**Integration Points:**
- Fires `attendance.daily_summary` event → consumed by `performance-svc` and `billing-svc`
- Pulls approved leave from `leave.approved` event to avoid false absences

---

### 3. Performance / Scorecard (`performance-svc`)

**Purpose:** Continuous, data-driven performance measurement replacing periodic manual reviews.

**Key Entities:** `KPI`, `KPIAssignment`, `ScoreEntry`, `AppraisalCycle`, `GoalSheet`, `Competency`

**Core Workflows:**
- Define KPIs per role/department (quantitative and qualitative)
- Automated score ingestion from source systems (attendance, billing, ops SLAs)
- Weighted scorecard aggregation engine
- Appraisal cycle management (quarterly / annual)
- AI-powered: detect performance trends, recommend improvement actions, flag outliers

**Scoring Formula:**
```
kpi_score(i) = (actual_value / target_value) * weight(i)   -- for "higher is better" KPIs
kpi_score(i) = (target_value / actual_value) * weight(i)   -- for "lower is better" KPIs
overall_score = SUM(kpi_score(i)) / SUM(weight(i)) * 100
band = CASE
  WHEN overall_score >= 90 → 'Outstanding'
  WHEN overall_score >= 75 → 'Exceeds Expectations'
  WHEN overall_score >= 60 → 'Meets Expectations'
  WHEN overall_score >= 45 → 'Needs Improvement'
  ELSE                      → 'Unsatisfactory'
END
```

**Integration Points:**
- Subscribes to: `attendance.daily_summary`, `billing.target_vs_actual`, `ops.sla_report`
- Publishes: `performance.score_updated` → consumed by `finance-svc` for incentive calculations

---

### 4. MaxBill (`billing-svc`)

**Purpose:** End-to-end billing lifecycle — from usage capture to invoice delivery and collections.

**Key Entities:** `BillingAccount`, `PricingPlan`, `UsageRecord`, `Invoice`, `Payment`, `CreditNote`

**Core Workflows:**
- Usage metering and aggregation (daily batch + real-time)
- Pricing plan application (flat, tiered, volume, pay-per-use)
- Invoice generation and PDF rendering
- Payment reconciliation (manual, auto-debit, gateway webhook)
- Dunning / collections workflow with escalation tiers
- AI-powered: churn prediction on overdue accounts, anomaly detection on usage spikes

**Billing Cycle Engine:**
```
1. Collect usage_records for billing_period
2. Apply pricing_plan rules → line_items[]
3. Apply discounts / promotions
4. Compute tax (rule-based per jurisdiction)
5. Generate invoice (status: draft)
6. Send for approval if amount > threshold
7. Finalize → dispatch to customer
8. Monitor payment_due_date → trigger dunning if unpaid
```

**Integration Points:**
- Publishes: `billing.invoice_finalized`, `billing.payment_received`, `billing.target_vs_actual`
- Consumed by: `finance-svc` (revenue recognition), `performance-svc` (billing KPIs)

---

### 5. Finance (`finance-svc`)

**Purpose:** Authoritative financial record-keeping, reporting, and forecasting across the ecosystem.

**Key Entities:** `ChartOfAccounts`, `JournalEntry`, `GLTransaction`, `Budget`, `CostCentre`, `FinancialReport`

**Core Workflows:**
- Double-entry bookkeeping engine (every ecosystem event generates GL postings)
- Budget vs. actual tracking by cost centre and period
- Month-end / year-end close checklist automation
- Financial report generation: P&L, Balance Sheet, Cash Flow
- AI-powered: cash flow forecasting, budget variance explanation, anomaly flagging in GL

**Auto-Posting Rules (examples):**
```yaml
trigger: billing.payment_received
debit:  accounts_receivable   # reverse AR
credit: cash_and_equivalents  # recognize cash

trigger: asset.depreciation_run
debit:  depreciation_expense
credit: accumulated_depreciation

trigger: attendance.overtime_approved
debit:  salary_expense_overtime
credit: accrued_liabilities
```

**Integration Points:**
- Subscribes to ALL module events that have financial implications
- Publishes: `finance.budget_alert` when variance > threshold → consumed by `ops-svc`

---

### 6. Operations (`ops-svc`)

**Purpose:** Orchestrate cross-module workflows, monitor SLAs, and surface the command-and-control dashboard.

**Key Entities:** `Workflow`, `WorkflowStep`, `Task`, `SLAPolicy`, `Alert`, `ResourcePool`

**Core Workflows:**
- Visual workflow builder (DAG-based) for multi-step approvals and automations
- Task assignment with priority, deadline, and escalation rules
- SLA policy enforcement with breach prediction
- Unified alert aggregation from all modules
- AI-powered: predict bottlenecks, auto-reassign tasks, suggest process optimisations

**Integration Points:**
- Acts as the **orchestration hub** — subscribes to all `*.alert` and `*.escalation` events
- Exposes the **unified ops dashboard** (real-time metrics from all modules)
- Triggers cross-module remediation workflows on threshold breaches

---

## Shared Infrastructure

### Authentication & RBAC
```
Every request carries a JWT with:
  - sub (user_id)
  - org_id
  - roles[]          -- e.g. ["finance.viewer", "asset.admin"]
  - scope[]          -- module-level permission scopes

Permission check pattern:
  require_permission("asset:write")
  require_permission("billing:invoice:approve")
```

### Shared Entity Registry
All modules reference these canonical entities by UUID — never store duplicates:
- `organisations` — root tenant
- `employees` — shared person record
- `departments` — org chart nodes
- `locations` — physical/logical locations
- `cost_centres` — financial allocation units

### Event Schema Convention
```json
{
  "event_id": "uuid-v4",
  "event_type": "<module>.<entity>.<action>",
  "source_service": "attendance-svc",
  "org_id": "uuid",
  "timestamp": "ISO-8601",
  "version": "1.0",
  "payload": { /* domain-specific data */ },
  "correlation_id": "uuid"  // trace across modules
}
```

### API Conventions
- **Versioning:** `/api/v1/...`
- **Pagination:** cursor-based (`?cursor=&limit=`)
- **Filtering:** `?filter[status]=active&filter[category_id]=...`
- **Sorting:** `?sort=-created_at`
- **Error format:**
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Human-readable description",
    "details": [{ "field": "email", "issue": "required" }],
    "trace_id": "uuid"
  }
}
```

---

## Development Workflow

### When asked to build a new module or feature, Manus must:

1. **Clarify scope** — which module(s), which entities, which workflows?
2. **Check shared entities** — does this require a new entity or reference an existing one?
3. **Design the data schema** — write migration SQL first
4. **Define events** — what does this module emit and consume?
5. **Scaffold the service** — use the standard project structure below
6. **Write the core logic** — business rules before API layer
7. **Add integration hooks** — event publisher/subscriber wiring
8. **Write tests** — unit tests for business logic, integration tests for API endpoints
9. **Document** — update API spec and event catalog

### Standard Project Structure
```
<module>-svc/
├── src/
│   ├── domain/          # Entities, value objects, domain services
│   ├── application/     # Use cases / commands / queries
│   ├── infrastructure/  # DB repos, event bus adapters, external APIs
│   ├── api/             # HTTP controllers, route definitions
│   └── shared/          # DTOs, validators, error types
├── migrations/          # Timestamped SQL migration files
├── tests/
│   ├── unit/
│   └── integration/
├── Dockerfile
├── openapi.yaml         # API contract
└── events.yaml          # Published/subscribed event catalog
```

---

## AI Layer Integration

When adding AI capabilities to any module, follow this pattern:

```
1. Define the AI task:
   - Input features (from existing data)
   - Prediction target
   - Acceptable latency (real-time vs. batch)

2. Choose inference mode:
   - Real-time: embed model call in service request path (< 200ms budget)
   - Batch: nightly job → write predictions to predictions table
   - Streaming: consume events → update predictions incrementally

3. Expose predictions:
   - Store in predictions table with model_version + confidence_score
   - Surface via API as advisory (never block user actions on prediction)
   - Feed back corrections as training signal

4. Monitor:
   - Track prediction accuracy weekly
   - Alert when accuracy drops > 5% vs. baseline
   - Gate model updates behind A/B comparison
```

---

## Migration Strategy

| Phase | Activity | Success Criteria |
|---|---|---|
| **Phase 0** | Data audit & cleansing of manual records | >90% records mapped to canonical entities |
| **Phase 1** | Shadow mode — AI modules run in parallel with manual | Module outputs match manual within 5% |
| **Phase 2** | Assisted mode — AI outputs are primary, human confirms exceptions | <10% exception rate |
| **Phase 3** | Autonomous mode — AI handles routine; humans handle edge cases | <2% exception rate |
| **Phase 4** | Optimisation — continuous learning loop, cross-module intelligence | Measurable efficiency gain vs. baseline |

---

## Quick-Reference Cheat Sheet

| Task | Where to look |
|---|---|
| Add a new KPI type | `performance-svc/src/domain/kpi.ts` + migration |
| Change billing calculation | `billing-svc/src/domain/pricing-engine.ts` |
| Add a new GL auto-posting rule | `finance-svc/src/infrastructure/posting-rules.yaml` |
| Create a new approval workflow | `ops-svc` workflow builder API |
| Add a new asset category with custom fields | `asset-svc` category config + `metadata` JSONB schema |
| Wire a new event consumer | Implement `IEventHandler` in the target service |
| Add a new RBAC permission | `auth-svc/src/permissions-registry.ts` |

---

## Notes for Manus

- **Always emit events** when state changes — downstream modules depend on them.
- **Never hardcode org_id** — the system is multi-tenant from day one.
- **Prefer database constraints** over application-level validation for data integrity.
- **When in doubt about business rules**, ask the user to confirm before implementing — these rules drive money and compliance.
- **Keep modules decoupled** — if Module A needs data from Module B, it should either subscribe to events or call a well-defined API, never query Module B's database directly.
- **Performance baseline:** all read APIs must respond < 300ms at p95 under normal load; batch jobs must complete within their scheduled window.
