# Risk Agent — Test Scenarios

Run these in Cursor or Claude Code in Agent mode. Each scenario tells the agent:
- which agent file to read
- what context to use
- what to produce

Before running any scenario: confirm the Configuration Profile is loaded.
If not, run Scenario 0 first.

---

## Context Awareness Rule (applies to all scenarios)

Before executing any scenario, the agent should assess what it already knows:
- Has the Configuration Profile been loaded this session? If yes, skip re-loading it.
- Has the scoring scale already been read via get_object_schema? If yes, use it from context.
- Has the risk record already been read? If yes, use the values already in context — do not re-fetch.

The agent should state explicitly at the start of each run: "Here is the context I already have: [list]. Here is what I need to fetch: [list]."

---

## Scenario 0 — Configuration Profile Onboarding (Customised Instance)

**Use this when:** Testing against a non-OOTB customer who has renamed fields and roles.

```
Read CLAUDE.md and .claude/agents/configuration-agent.md.

The following customer configuration exists. Run the Profile Builder and produce
a complete Configuration Profile object for this customer:

Customer: Meridian Financial Services
Roles configured:
  - "GRC Lead" — has full CRUD on all risks and controls (equivalent to Risk Manager)
  - "Risk Custodian" — owns assigned risks, can create assessments, cannot delete
  - "Review Analyst" — can complete assessments, no risk ownership
  - "Controls Specialist" — owns controls, can create control assessments

Workflow statuses on their Risk object:
  Draft → Active → Under Assessment → Monitored → Escalated → Closed → Legacy

Field names on their Risk object (discovered via get_object_schema):
  - "risk_name" (not "name")
  - "gross_risk_score" (their inherent score)
  - "net_risk_score" (their residual score)
  - "risk_appetite_level" (custom field — they want this used for monitoring)
  - "assigned_to" (not "owner_id")
  - "business_unit" (not "org_unit_id")
  - "last_review_date" (not "last_assessment_date")
  - "next_review_due" (not "assessment_due_date")

Monitoring preference: They want tiered monitoring.
  - net_risk_score > risk_appetite_level = amber warning
  - net_risk_score > gross_risk_score = red breach
  (They do not have a separate tolerance field — gross score acts as hard ceiling)

Produce the full Configuration Profile JSON and explain each mapping decision.
Show what monitoring mode was selected and why.
```

**What to verify:**
- [ ] Profile uses "GRC Lead" for admin_equivalent
- [ ] Profile uses "Risk Custodian" for risk_owner_equivalent
- [ ] Profile uses "Review Analyst" for assessor_equivalent
- [ ] terminal_statuses includes "Closed" and "Legacy"
- [ ] field_mapping uses "gross_risk_score" for inherent, "net_risk_score" for residual
- [ ] field_mapping uses "risk_appetite_level" for risk_appetite_field
- [ ] monitoring_rule.mode = "tiered_appetite_tolerance" with amber at appetite, red at inherent
- [ ] Profile is versioned and ready to emit configuration_changed event

---

## Scenario 1 — First Assessment Baseline (OOTB Config)

**Purpose:** Confirm the agent correctly identifies first assessments and sets baseline only — no breach alert fired.

```
Read CLAUDE.md and .claude/agents/operations-agent.md.
Read .claude/skills/risk-monitoring/SKILL.md for comparison logic.

Context (treat as already loaded — do not re-fetch):
  Configuration Profile: OOTB defaults (Risk Manager, Risk Owner, Risk Assessor roles.
    Monitoring mode: baseline_only. inherent_score_field: "inherent_risk_score",
    residual_score_field: "residual_risk_score". terminal_statuses: ["Archived"])

Scenario: Assessment Agent has just fired a score_applied event with:
  risk_id: RISK-089
  risk_name: "Vendor Payment Fraud"
  assessment_cycle: "first"
  inherent_risk_score: "High"
  residual_risk_score: null
  scoring_scale: [
    {label:"Very Low",rank:1},{label:"Low",rank:2},{label:"Medium",rank:3},
    {label:"High",rank:4},{label:"Very High",rank:5}
  ]
  applied_by: "agent"
  applied_at: "2026-04-14T09:00:00Z"

Execute Job 3 (Continuous Monitoring). Show your reasoning step by step.
```

**What to verify:**
- [ ] Agent correctly identifies assessment_cycle = "first"
- [ ] Agent sets baseline — no comparison runs
- [ ] Agent emits MON-001 (baseline set notification) to Risk Owner
- [ ] NO breach alert fired (MON-003 must NOT appear)
- [ ] AuditLog entry written: actor=agent, action=baseline_set, no human_decision field needed
- [ ] Agent states clearly: "This is the first assessment. Baseline set. Monitoring will begin from next assessment."

---

## Scenario 2 — Breach Detection and Full Issue Creation Chain

**Purpose:** Test the complete breach response from score comparison through to Issue created and assigned.

```
Read CLAUDE.md and .claude/agents/operations-agent.md.
Read .claude/skills/risk-monitoring/SKILL.md and .claude/skills/human-interaction/SKILL.md.

Context already loaded:
  Configuration Profile: OOTB defaults

score_applied event received:
  risk_id: RISK-052
  risk_name: "Supplier Concentration Risk"
  assessment_cycle: "subsequent"
  inherent_risk_score: "Medium"
  residual_risk_score: "High"
  scoring_scale: [Very Low=1, Low=2, Medium=3, High=4, Very High=5]
  workflow_status: "Assessment"

Risk record additional context (already read):
  owner_id: USR-042 (Sarah Chen, role: Risk Owner)
  org_unit_id: OU-007 (EMEA Procurement)
  category: "Third Party Risk"
  linked_controls: [CTL-019 (Supplier Diversification Review, Healthy),
                    CTL-023 (Concentration Limit Policy, Partial)]

Available Issue Managers in EMEA Procurement / Third Party Risk category:
  USR-088 (James Wright, 2 open issues — workload: low)
  USR-091 (Ana Costa, 5 open issues — workload: medium)

Execute Jobs 3 and 4 (Monitoring then Breach Response).

For Job 4: produce the complete ops.issue_plan HandoffCard in A2UI JSON format.
Then simulate: Risk Owner approves the plan as-is.
Show all resulting actions the agent takes and AuditLog entries written.
```

**What to verify:**
- [ ] Ordinal comparison correctly identifies High (rank 4) > Medium (rank 3) = BREACH
- [ ] MON-003 alert fired with workflow_status "Assessment" included in message
- [ ] ops.issue_plan HandoffCard rendered in correct A2UI JSON format
- [ ] Proposed severity = "High" (1 level delta)
- [ ] Proposed issue manager = USR-088 James Wright (lower workload)
- [ ] Plan steps list exactly 4 actions
- [ ] Agent_reasoning section explains why James Wright was chosen
- [ ] On approval: Issue created, linked to RISK-052, assigned to James Wright
- [ ] AuditLog entry: agent_recommendation shows proposed fields, human_decision = "approved_as_is"
- [ ] BRE-002 notification to Sarah Chen, BRE-003 to James Wright

---

## Scenario 3 — Breach Detection With Customised Config (Appetite Threshold)

**Purpose:** Test monitoring rule engine with non-OOTB monitoring mode.

```
Read CLAUDE.md and .claude/agents/operations-agent.md.
Read .claude/skills/risk-monitoring/SKILL.md.

Context already loaded:
  Configuration Profile: Meridian Financial Services (from Scenario 0)
  - Monitoring mode: tiered_appetite_tolerance
  - inherent = gross_risk_score, residual = net_risk_score, appetite = risk_appetite_level
  - Field names: risk_name, assigned_to, business_unit, next_review_due

score_applied event:
  risk_id: RISK-031
  risk_name: "Regulatory Reporting Failure"
  assessment_cycle: "subsequent"
  gross_risk_score: "High"     (inherent)
  net_risk_score: "Medium"     (residual)
  risk_appetite_level: "Low"   (appetite threshold)
  scoring_scale: [Low=1, Medium=2, High=3, Very High=4]

Run the Monitoring Rule Engine. Determine:
1. Is net_risk_score > risk_appetite_level? (amber breach condition)
2. Is net_risk_score > gross_risk_score? (red breach condition)
3. What notification fires and at what severity?
4. Produce the correct alert using the customer's field names throughout.
```

**What to verify:**
- [ ] Agent reads monitoring_rule.mode from profile (tiered_appetite_tolerance), not hardcoded
- [ ] Agent reads field names from profile.field_mapping, not hardcoded
- [ ] Medium (rank 2) > Low (rank 1) = amber breach condition TRUE
- [ ] Medium (rank 2) > High (rank 3) = red breach condition FALSE
- [ ] MON-003-AMBER fires (warning, not full breach)
- [ ] Alert message uses customer's field names: "net_risk_score (Medium) exceeds risk_appetite_level (Low)"
- [ ] Alert does NOT say "residual_risk_score" or "inherent_risk_score" — those are OOTB names

---

## Scenario 4 — CTRL-6 Chain With Active Assessment Paused

**Purpose:** Test CTRL-6 chain triggering, Assessment Agent pause, and consolidated HandoffCard.

```
Read CLAUDE.md and .claude/agents/operations-agent.md.
Read .claude/skills/risk-monitoring/SKILL.md.

Context already loaded:
  Configuration Profile: OOTB defaults

control_score_applied event received from Risk Maestro:
  control_id: CTL-019
  control_name: "Supplier Diversification Review"
  control_status: "Key Control"
  design_effectiveness: "Not Effective"
  control_effectiveness: "Exceptions Noted"
  linked_risk_ids: ["RISK-052", "RISK-031", "RISK-044"]

Additional context:
  RISK-052: workflow_status="Assessment", owner=USR-042, residual="High"
  RISK-031: workflow_status="Monitored", owner=USR-055, residual="Medium", has ACTIVE Assessment Agent cycle
  RISK-044: workflow_status="Closed"  ← this should be treated as terminal_status equivalent

Execute Job 8 (CTRL-6 Chain). Show:
1. Which risks are processed and which are skipped (and why)
2. The CTRL-6 failure matrix evaluation
3. The consolidated HandoffCard (one card, not three)
4. The breach_priority_lock event sent to Assessment Agent for RISK-031
5. The escalation to Risk Manager since this is a Key Control
```

**What to verify:**
- [ ] RISK-044 SKIPPED — "Closed" is in terminal_statuses
- [ ] RISK-052 and RISK-031 processed
- [ ] CTRL-6 matrix: Not Effective + Exceptions Noted = FAILING (not CRITICAL)
- [ ] ONE consolidated CTL-003 HandoffCard listing RISK-052 and RISK-031
- [ ] breach_priority_lock event sent for RISK-031 (has active Assessment cycle)
- [ ] Key Control status triggers escalation to Risk Manager (not just Risk Owners)
- [ ] T3 ops.ctrl6_card generated per affected risk (2 cards, not 3)
- [ ] RISK-044 not mentioned in any notification

---

## Scenario 5 — Breach Re-fires on Risk With Open Issue (Action Item Path)

**Purpose:** Test Job 4b — second breach on same risk → Action Item not new Issue.

```
Read CLAUDE.md and .claude/agents/operations-agent.md.

Context already loaded:
  Configuration Profile: OOTB defaults

score_applied event:
  risk_id: RISK-052
  assessment_cycle: "subsequent"
  inherent_risk_score: "Medium"
  residual_risk_score: "High"

Additional context (already read from MCP):
  Existing open Issue on RISK-052:
    issue_id: ISS-023
    title: "Supplier Concentration Risk — Residual breach 2026-03-01"
    status: "In Remediation"
    owner_id: USR-088 (James Wright)
    mitigation_plan: "Diversify supplier base across 3 additional vendors"
    actions_defined: 2
    actions_complete: 0

Execute Job 4 (Breach Response).
The agent should detect the open Issue and route to Job 4b.
Produce the ops.existing_issue_action HandoffCard.
Then simulate: Risk Owner approves.
Show all actions taken.
```

**What to verify:**
- [ ] Agent checks list_relationships before creating any Issue
- [ ] Agent detects ISS-023 as open Issue
- [ ] NO new Issue created — agent routes to Job 4b
- [ ] Update comment written to ISS-023: "Breach re-detected. Score still High vs Inherent Medium."
- [ ] ops.existing_issue_action HandoffCard shows: existing issue ID, previous score, current score
- [ ] On approval: Action Item created and linked to ISS-023
- [ ] AuditLog shows: action="action_item_added", linked_to_existing_issue=ISS-023

---

## Scenario 6 — Risk Owner Account Deactivated Mid-Cycle

**Purpose:** Test the deactivated user handoff protocol.

```
Read CLAUDE.md and .claude/agents/operations-agent.md.

Context already loaded:
  Configuration Profile: OOTB defaults
  admin_equivalent role_ids: [ROLE-001] → Risk Manager role

The following situation has occurred:
  USR-042 (Sarah Chen, Risk Owner) has been deactivated in the system.
  She had the following open items:
    - 2 open HandoffCards in Needs You: ops.issue_plan for RISK-052 and ops.ctrl6_card for RISK-044
    - She is the named owner of RISK-052, RISK-031, RISK-055 (3 risk records)
    - She was the assigned assessor on REA-019 (in progress, 3 days until deadline)

The agent detects the deactivation via an audit log entry:
  actor: "admin", action: "user_deactivated", user_id: "USR-042"

Execute the deactivated user handoff protocol. Show:
1. What the agent does with the open HandoffCards
2. What the agent does with the 3 risk records she owned
3. What the agent does with the in-progress assessment
4. The notification sent to Admin
5. The reassignment prompt presented to Admin
6. All AuditLog entries
```

**What to verify:**
- [ ] Agent scans for all items linked to deactivated user
- [ ] HandoffCards reassigned to admin_equivalent (Risk Manager role holders)
- [ ] Risk records: owner field flagged as "unassigned" — agent does NOT auto-assign
- [ ] In-progress assessment: same SLA monitoring continues, Admin notified to reassign assessor
- [ ] Admin receives consolidated notification listing ALL affected items
- [ ] Admin presented with reassignment options per item (one HandoffCard per item type)
- [ ] AuditLog: actor=agent, action=user_deactivation_handoff, user_id=USR-042, items_affected=[list]
- [ ] No items lost — all preserved and visible to Admin

---

## Scenario 7 — Full Context Awareness Test

**Purpose:** Test the agent's ability to assess and use its existing context efficiently.

```
Read CLAUDE.md and .claude/agents/operations-agent.md.

You have just completed Scenario 2 in this session.
The following is already in your context from that run:
  - Configuration Profile (OOTB defaults)
  - Risk record RISK-052 (all fields)
  - Scoring scale [Very Low=1...Very High=5]
  - Available Issue Managers USR-088 and USR-091

A new score_applied event has arrived:
  risk_id: RISK-052
  assessment_cycle: "subsequent"
  inherent_risk_score: "Medium"
  residual_risk_score: "Medium"

Before executing: state explicitly what context you already have and what
(if anything) you need to re-fetch. Then execute Job 3.
```

**What to verify:**
- [ ] Agent explicitly states it already has RISK-052 record, scoring scale, and profile — no re-fetch
- [ ] Agent reads residual and inherent from context, not from new MCP call
- [ ] Medium (rank 3) vs Medium (rank 3) = NO CHANGE → Healthy
- [ ] MON-002 fires (Watching zone only — no email)
- [ ] Agent does NOT re-load Configuration Profile (already loaded)
- [ ] Agent statements: "Using scoring scale already in context. No re-fetch needed."

---

## Scenario 8 — Configuration Profile with Partial Appetite Fields

**Purpose:** Test graceful fallback when monitoring rule prerequisites are missing.

```
Read CLAUDE.md and .claude/agents/operations-agent.md.

Configuration Profile loaded:
  monitoring_rule.mode: "appetite_threshold"
  field_mapping.risk_appetite_field: "risk_appetite_level"

But risk record RISK-073 does not have risk_appetite_level populated (null).

score_applied event:
  risk_id: RISK-073
  risk_name: "IT Infrastructure Failure"
  assessment_cycle: "subsequent"
  inherent_risk_score: "Medium"
  residual_risk_score: "High"

Execute Job 3. Show what the agent does when the threshold field is null for this specific risk.
The agent should:
1. Detect that risk_appetite_level is null for RISK-073
2. Not silently skip — surface a notification
3. Offer the customer a choice
4. Execute whatever path the customer chooses
```

**What to verify:**
- [ ] Agent detects risk_appetite_level = null before running comparison
- [ ] Agent does NOT silently fall back to baseline — fires a notification first
- [ ] Notification states: "Risk RISK-073 is configured for appetite-threshold monitoring but the appetite field is empty. Should I: (a) Monitor against inherent score instead for this risk, or (b) Skip monitoring until appetite is filled in?"
- [ ] Agent waits for customer choice (T3 decision point)
- [ ] If customer chooses (a): runs baseline comparison, fires breach alert (High > Medium)
- [ ] If customer chooses (b): skips silently, logs to AuditLog with reason

---

## Manual Testing Checklist

For each scenario, verify:

**Correctness:**
- [ ] Agent reads correct files before executing (states what it read)
- [ ] Agent states context it already has vs what it needs to fetch
- [ ] All field names read from Configuration Profile, not hardcoded
- [ ] All role resolution via profile.role_mapping, not string matching
- [ ] Ordinal score comparison uses ranking, never string comparison

**Safety:**
- [ ] No write operations executed without T3 approval
- [ ] No MON-003 fired on first assessments
- [ ] No breach alert fired on terminal_status risks
- [ ] Deactivated user items transferred, not lost
- [ ] AuditLog written on every agent action

**Output quality:**
- [ ] A2UI JSON is valid and complete for all HandoffCards
- [ ] Notification IDs correct (MON-001, MON-003, BRE-002 etc.)
- [ ] Agent reasoning section explains why each decision was made
- [ ] Counter-checks fire on modify paths where conditions are met
