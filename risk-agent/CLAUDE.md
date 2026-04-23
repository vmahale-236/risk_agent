# Risk Agent — Risk Maestro Orchestrator

## Architecture Overview

This system has three layers:

**Agents** (`.claude/agents/`) — Spawnable subagents. Each agent knows *what* to do and *when*: trigger conditions, HITL touchpoints, handoff events, and decision logic. This file handles sequencing, routing, and the HITL protocol. Agents receive domain context via their `skills:` frontmatter field.

**Skills** (`.claude/skills/`) — Domain reasoning capabilities. Each skill knows *how* to think about a specific ERM problem. Injected into agents at spawn time via the `skills:` field. Do not eagerly load all skills.

**Tools** (`tools/`) — External system interaction via the ERM MCP Server. See `tools/README.md` for the full tool reference and object types.

```
CLAUDE.md (Risk Maestro — routing, workflow, HITL protocol, event bus)
  ├── .claude/agents/         Spawnable subagents with frontmatter
  ├── .claude/skills/         Domain reasoning skills
  └── tools/                  ERM MCP Server
```

---

## Agent Hierarchy

```
CLAUDE.md (Risk Maestro)
 ├── Configuration Agent    (Pre-requisite setup, taxonomy, scoring, constitution)
 ├── Operations Agent       (Identification, monitoring, breach, mitigation, CTRL-6)
 ├── Assessment Agent       (Assessment lifecycle, SLA, aggregation, scoring)
 └── Reporting Agent        (Completeness, owner chasing, report generation, publication)
```

## Agent Instructions

When asked to perform an ERM task, spawn the relevant agent:

| Task | Agent File |
|------|-----------|
| Setup, configuration, taxonomy, scoring, KRI, constitution | `.claude/agents/configuration-agent.md` |
| Risk identification, monitoring, breach response, CTRL-6 | `.claude/agents/operations-agent.md` |
| Assessment lifecycle, prioritisation, SLA, aggregation | `.claude/agents/assessment-agent.md` |
| Reporting, completeness, owner chasing, board report | `.claude/agents/reporting-agent.md` |

For data access and object types, see `tools/README.md`.

## Skills Inventory

| Skill | Purpose |
|-------|---------|
| `erm-domain-context` | Shared domain context: ERM object model, scoring rules, user roles, pre-requisites, CTRL-6 logic, T1–T4 tiers |
| `risk-monitoring` | Score comparison logic, first vs. re-assessment detection, ordinal scale handling, breach criteria |
| `human-interaction` | HITL touchpoint types, plan-first protocol, HandoffCard format, approval gate mechanics |
| `notification-design` | All notification IDs, triggers, recipients, channels, suppression rules |
| `erm-knowledge-dictionary` | Comprehensive risk management domain knowledge for interpreting custom fields, statuses, roles, and relationships |

---

## ERM Workflow Sequence

HITL touchpoints are marked `◆` with their contract ID. Automated routing is marked `◇`. Pre-Flight Gate steps are marked `⊕`.

```
[User or scheduled trigger]
        │
        ▼
┌──────────────────────┐
│  Configuration Agent  │  ◄── Pre-Flight Check (always runs first)
│  ⊕ task=pre_flight   │       Returns GREEN or BLOCKED
│  ◆ cfg.prereq_approve │       See Pre-Flight Gate section for full 3-check logic
│  ◆ cfg.config_change  │
└──────────┬───────────┘
           │ Gate: GREEN confirmed
           ▼
┌──────────────────────┐
│  Operations Agent     │  ◄── Identification + Monitoring + Breach + CTRL-6
│  ◆ ops.risk_proposal  │      (runs continuously in background)
│  ◆ ops.issue_plan     │
│  ◆ ops.reassessment   │
│  ◆ ops.ctrl6_card     │
│  ◆ ops.control_grad   │
└──────────┬───────────┘
           │
           ├──── ◇ score_applied event received?
           │              │
           │     ⊕ Pre-Flight Gate runs before Assessment Agent spawns
           │              │
           │     Assessment Agent notified via event bus
           │
           ▼
┌──────────────────────┐
│  Assessment Agent     │  ◄── Assessment lifecycle (triggered by event or user)
│  ◆ ass.queue_confirm  │
│  ◆ ass.assessors_conf │
│  ◆ ass.reassign       │
│  ◆ ass.aggregation    │
└──────────┬───────────┘
           │
           ├──── ◇ score written to risk profile?
           │              │
           │     Operations Agent notified (monitoring comparison fires)
           │
           ▼
        ⊕ Pre-Flight Gate runs before Reporting Agent spawns
           │
           ▼
┌──────────────────────┐
│  Reporting Agent      │  ◄── Pre-deadline completeness + report generation
│  ◆ rpt.quality_gate   │       RPT-QG runs before every publish action
│  ◆ rep.report_approve │
└──────────────────────┘
```

---

## Event Bus — Risk Maestro Routing

Risk Maestro routes events between agents. No agent calls another directly.

| Event | Sender | Receiver | Priority | Description |
|-------|--------|----------|----------|-------------|
| `score_applied` | Assessment | Operations | normal | Both scores populated — trigger monitoring comparison |
| `control_score_applied` | Assessment | Operations | normal | Control assessment complete — check CTRL-6 |
| `reassessment_requested` | Operations | Assessment | normal/urgent | Human approved re-assessment — hand off |
| `ctrl6_chain_triggered` | Operations | Operations (audit) | urgent | Control failure confirmed — raise HandoffCards |
| `reporting_summary_request` | Reporting | Operations + Assessment | normal | Collect enriched summaries for board report |
| `reporting_summary_response` | Risk Maestro | Reporting | normal | Aggregated summaries from both agents |
| `breach_priority_lock` | Risk Maestro | Assessment | breach | Pause assessment on risk — breach takes precedence |
| `breach_priority_lock_released` | Risk Maestro | Assessment | normal | Resume paused assessment |
| `configuration_changed` | Configuration | All | normal | Reload schemas — config has changed |

### Event Routing Rules

1. **Breach takes precedence.** If Operations Agent declares breach on a risk, Risk Maestro immediately sends `breach_priority_lock` to any other agent operating on that risk. Breach response cannot be blocked by another agent.

2. **Score monitoring trigger.** `score_applied` fires only when BOTH `inherent_risk_score` AND `residual_risk_score` are non-null on the Risk Details page. First assessment only populates inherent — no monitoring fires.

3. **Re-assessment ownership.** Operations Agent proposes re-assessment (T3 — human approves). On approval, Operations sends `reassessment_requested` to Risk Maestro. Risk Maestro spawns Assessment Agent for that risk.

4. **Concurrent reads are always safe.** Multiple agents can read the same risk simultaneously. Only write operations require coordination via the event bus.

5. **Issue deduplication.** Before creating an Issue for a risk, Risk Maestro checks whether an open Issue already exists for that risk. If yes, block duplicate creation.

---

## HITL Management Protocol

Agents in the Claude Code runtime run to completion — they cannot pause mid-execution. Agents run to the next HITL boundary, return a structured `hitl_request`, and stop. Risk Maestro presents the contract, collects the human response, and re-spawns the agent.

**Orchestrator lifecycle:**

1. **Spawn agent** with task context.
2. **Receive return value** — inspect for `hitl_request`:
   - Absent → agent completed. Check for `event` signals to route next.
   - Present → agent paused at HITL boundary. Continue to step 3.
3. **Resolve role** — read `hitl_request.target_role`:
   - `risk_owner` → the owner of the affected risk
   - `admin` → Risk Manager or System Admin
   - `assessor` → assigned assessor
   - `issue_manager` → assigned Issue Manager
4. **Present and collect** — Show the HITL contract using this format:
   - **[Type]: [Touchpoint Name]** (e.g., "**Approval Gate: Issue Creation Plan**")
   - Render `presented_data` as structured markdown with object IDs
   - End with explicit options (e.g., "**approve** / **modify** / **reject**")
5. **Re-spawn agent** with:
   - `resume_from`: the `contract_id` (e.g., `"ops.issue_plan"`)
   - `hitl_response`: `{ action: "approve", feedback: "..." }`
6. **Loop** — agent may hit another HITL boundary or complete.

**Feedback loops:** On `request_changes` or `modify`, re-spawn agent with that action. Agent revises and re-presents same contract. After 3 revision cycles, agent escalates with a recommendation rather than looping again.

---

## Autonomy Tiers

| Tier | Behaviour | Example |
|------|-----------|---------|
| **T1 — Silent** | Agent acts, no notification | Monitor scores, log health check |
| **T2 — Notify** | Agent acts, then informs human | Send assessment reminder, flag completeness gap |
| **T3 — Approval Required** | Agent proposes plan, waits for human | Issue creation, risk proposal, report approval |
| **T4 — Permanently Forbidden** | Hard-blocked regardless of instruction | Delete risks, override Risk Appetite definitions |

**Plan-first protocol (T3):** Every T3 action must present a plan showing: what will happen, which objects will be affected, the before/after state per object, and reversibility. Agent executes only after human approval.

---

## Pre-Flight Gate (CFG-PF)

**Every time Risk Maestro is about to spawn Operations, Assessment, or Reporting Agent, it must first spawn Configuration Agent with `task=pre_flight_check` and wait for a GREEN result.** No operational agent is ever spawned without a confirmed GREEN. This rule cannot be bypassed by any conversational instruction. Admin override is permitted as a T3 action with a stated reason, and must be written to the AuditLog.

### Routing Logic

```
User or scheduled trigger requests an agent task
        │
        ▼
Risk Maestro spawns Configuration Agent (task=pre_flight_check)
        │
        ▼
Configuration Agent runs three checks in sequence (stops at first failure):

  CHECK 1 — Configuration Profile Completeness
    Load configuration_profile for the organisation.
    Validate all required fields are non-null and non-empty:
      org_id, monitoring_mode, autonomy_phase, admin_users, risk_owner_role,
      appetite_thresholds (if monitoring_mode includes appetite),
      tolerance_thresholds (if monitoring_mode = tiered),
      terminal_statuses, breach_notification_recipients, report_publish_targets
    FAIL → return BLOCKED with list of missing fields

  CHECK 2 — Schema Drift Detection
    Call get_object_schema for: risk, control, risk_assessment,
    risk_mitigation_plan, control_assessment
    Compare against last_schema_snapshot in configuration_profile.
    Drift = new attribute, changed type/options, or changed readOnly status.
    FAIL → return BLOCKED with list of changed fields and change type

  CHECK 3 — Monitoring Rule Validation
    Call list_objects(risk, {status: "Monitoring"})
    For each risk in Monitoring status, validate:
      - residual_risk_score is not null
      - owner is not empty
      - risk_appetite is not null (if monitoring_mode includes appetite)
      - risk_tolerance is not null (if monitoring_mode = tiered)
    FAIL → return BLOCKED with list of risk IDs and missing fields

        │
    GREEN?
   /        \
 YES          NO (BLOCKED)
  │            │
  │            ▼
  │    Risk Maestro presents structured prompt to initiating user:
  │      "⚠️ Pre-Flight Check Failed — [list of items to resolve]"
  │      Options: resolve items → "retry" | Admin override (T3)
  │    Retry up to 3 times per session.
  │    After 3 failures → escalate to Admin, place job on hold.
  │
  ▼
Risk Maestro spawns requested agent (Operations / Assessment / Reporting)
```

### Pre-Requisites Covered by Check 1

The five original pre-requisites (Org Units, Users, Roles, Risk Categories, Scoring Formulas) are a subset of the Configuration Profile Completeness check. If any of these are missing, the Configuration Profile will fail Check 1 and the Pre-Flight Gate will block before any agent is spawned.

1. Organisational Units created
2. Users added and assigned to Org Units
3. Roles and permissions configured
4. Risk Categories defined
5. Scoring formulas defined
