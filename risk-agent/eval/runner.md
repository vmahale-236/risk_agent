# Risk Agent — Evaluation Harness

Automated evaluation framework for Risk Agent. Runs after every sprint release.
Validates agent outputs against acceptance criteria without human review of each run.

---

## Structure

```
eval/
  runner.md               # How to run evaluations
  fixtures/
    ootb-profile.json     # OOTB configuration profile
    meridian-profile.json # Customised profile (Scenario 0)
    risks.json            # Test risk records
    users.json            # Test user directory
    scoring-scales.json   # Test scoring formulas
  assertions/
    monitoring.assertions.md
    breach-response.assertions.md
    assessment.assertions.md
    configuration.assertions.md
    edge-cases.assertions.md
```

---

## How to Run

In Cursor or Claude Code (Agent mode):

```
Read eval/runner.md and all files in eval/fixtures/.
Run the evaluation suite for [monitoring|breach-response|assessment|configuration|all].
For each assertion, output: PASS / FAIL / WARN and explain why.
Do not stop on first failure — run all assertions and produce a full report.
```

---

## Fixture: OOTB Configuration Profile
See `eval/fixtures/ootb-profile.json`

Key values for assertions:
- monitoring_mode: baseline_only
- terminal_statuses: ["Archived"]
- inherent_field: "inherent_risk_score"
- residual_field: "residual_risk_score"
- risk_owner_role_ids: ["ROLE-002"]
- admin_role_ids: ["ROLE-001"]

---

## Assertions: Monitoring

**ASSERT-MON-001: First assessment produces no breach alert**
```
Input:
  score_applied event: assessment_cycle="first", inherent="High", residual=null
  Profile: ootb-profile

Expected:
  - MON-001 notification fires (baseline set)
  - MON-003 does NOT fire
  - AuditLog entry: action=baseline_set
  - No Issue creation proposed

Fail condition: MON-003 fires on first assessment
```

**ASSERT-MON-002: Healthy subsequent assessment produces no breach**
```
Input:
  score_applied event: assessment_cycle="subsequent",
    inherent="High" (rank 4), residual="Medium" (rank 3)
  Profile: ootb-profile

Expected:
  - MON-002 fires (Watching zone update)
  - MON-003 does NOT fire
  - No Issue creation proposed

Fail condition: MON-003 fires when residual < inherent
```

**ASSERT-MON-003: Breach correctly detected**
```
Input:
  score_applied event: assessment_cycle="subsequent",
    inherent="Medium" (rank 3), residual="High" (rank 4)
  Profile: ootb-profile

Expected:
  - MON-003 fires
  - Breach alert includes workflow_status
  - ops.issue_plan HandoffCard generated

Fail condition: No breach detected when residual > inherent
```

**ASSERT-MON-004: Terminal status risk skipped**
```
Input:
  score_applied event for RISK-099 with workflow_status="Archived",
    inherent="Low", residual="Very High"
  Profile: ootb-profile

Expected:
  - Risk skipped silently
  - No MON-003
  - AuditLog: action=skip_terminal_status

Fail condition: Any notification or proposal fires for archived risk
```

**ASSERT-MON-005: Custom field monitoring (appetite threshold)**
```
Input:
  score_applied event: assessment_cycle="subsequent"
    risk has: net_risk_score="Medium", risk_appetite_level="Low", gross_risk_score="High"
  Profile: meridian-profile (tiered_appetite_tolerance mode)

Expected:
  - Amber warning fires (Medium > Low appetite)
  - Red breach does NOT fire (Medium < High inherent)
  - Alert uses customer field names: net_risk_score, risk_appetite_level

Fail condition: Wrong severity or OOTB field names used
```

**ASSERT-MON-006: Partial appetite field — fallback with notification**
```
Input:
  Profile: appetite_threshold mode configured
  Risk record: risk_appetite_level = null

Expected:
  - Agent fires notification: "appetite field empty, how should I proceed?"
  - Does NOT silently fall back to baseline
  - Does NOT fire breach alert before customer responds

Fail condition: Silent fallback without notification
```

---

## Assertions: Breach Response

**ASSERT-BRE-001: Issue creation plan uses correct field names**
```
Input:
  Breach detected on RISK-052 (OOTB profile)

Expected:
  - ops.issue_plan HandoffCard in valid A2UI JSON
  - All field references use ootb-profile field names
  - agent_reasoning section present and non-empty
  - reversibility section present
  - actions array has exactly 4 items

Fail condition: Missing sections, invalid JSON, wrong field names
```

**ASSERT-BRE-002: Existing issue → action item not new issue**
```
Input:
  Breach fires on RISK-052
  RISK-052 has open Issue ISS-023 (status: "In Remediation")

Expected:
  - No new Issue created
  - ops.existing_issue_action HandoffCard generated
  - Existing issue ID ISS-023 shown in HandoffCard

Fail condition: New Issue created when open Issue already exists
```

**ASSERT-BRE-003: Accept risk creates acceptance record**
```
Input:
  Risk Owner selects accept_risk on ops.issue_plan

Expected:
  - risk_acceptance object created with: accepted_by, accepted_at, accepted_score, review_date
  - review_date maximum 1 year from today
  - If residual in top 2 levels: escalate to admin for dual approval
  - MON-003 suppressed while acceptance is active

Fail condition: Risk acceptance created without mandatory reason or review date
```

**ASSERT-BRE-004: 3 rejections triggers escalation**
```
Input:
  ops.issue_plan rejected 3 times for same risk (revision_count = 3)

Expected:
  - Admin notified with all 3 prior proposals shown
  - Admin has options: approve / modify / accept_risk
  - HandoffCard badge: "Declined 3 times — escalated"

Fail condition: Agent creates Issue after 3 rejections without admin approval
```

**ASSERT-BRE-005: Modify path with counter-check**
```
Input:
  Risk Owner selects modify on ops.issue_plan
  Human changes severity from "High" to "Low"

Expected:
  - Counter-alert fires: "You've reduced severity from High to Low. This affects SLA. Confirm?"
  - Execution blocked until human confirms counter-alert
  - If confirmed: Issue created with Low severity
  - AuditLog: counter_alerts_shown=["severity_reduced"], human_confirmed_counter_alerts=true

Fail condition: Issue created at Low severity without counter-alert being shown
```

---

## Assertions: Assessment Lifecycle

**ASSERT-ASS-001: Empty score submission blocked**
```
Input:
  Assessment REA-045 submitted to workflow "Assessment" status
  All score fields: null

Expected:
  - score_applied event NOT fired
  - Notification: "Assessment submitted without scores. Cannot apply."
  - AuditLog: action=validate_fail, reason=empty_scores

Fail condition: score_applied event fires with null score values
```

**ASSERT-ASS-002: Conflict of interest detected**
```
Input:
  Assessor recommendation for RISK-052
  USR-042 is both the Risk Owner and a candidate assessor

Expected:
  - USR-042 flagged with conflict_of_interest=true
  - Not recommended as primary assessor
  - If human tries to assign USR-042: counter-alert fires

Fail condition: USR-042 assigned as assessor without conflict flag
```

**ASSERT-ASS-003: SLA reminder chain fires at correct thresholds**
```
Input:
  Assessment created with deadline in 10 days
  5 days pass with no submission

Expected:
  - At 50% (5 days): ASS-020 reminder to assessor only
  - NOT escalated to Risk Owner yet

  7.5 days pass:
  - At 75%: ASS-021 to assessor AND Risk Owner

  10 days pass:
  - At 100%: ASS-022 alert + ass.reassign HandoffCard

Fail condition: Wrong recipients at wrong thresholds
```

---

## Assertions: Configuration

**ASSERT-CFG-001: Profile built with correct OOTB defaults**
```
Input:
  New customer, no customisation, OOTB roles and field names

Expected:
  Profile contains:
  - admin_equivalent: "Risk Manager" role
  - risk_owner_equivalent: "Risk Owner" role
  - terminal_statuses: ["Archived"]
  - inherent_score_field: "inherent_risk_score"
  - monitoring_mode: "baseline_only"

Fail condition: Profile has null values for any OOTB default
```

**ASSERT-CFG-002: Schema drift detected and surfaced**
```
Input:
  Configuration Profile has field_mapping.risk_appetite_field = null
  get_object_schema returns a new field called "risk_appetite_threshold"
    (added by customer after profile was built)

Expected:
  - Agent detects field not in profile
  - Notification: "New field detected: risk_appetite_threshold. Should this be used for monitoring?"
  - T3 prompt to Admin to update profile

Fail condition: New field silently ignored
```

**ASSERT-CFG-003: Human direct settings change validated**
```
Input:
  Human directly renames "Archived" to "Closed" in Settings UI (not through Configuration Agent)
  Profile still has terminal_statuses: ["Archived"]
  Agent detects: workflow_status_change from audit log

Expected:
  - Agent validates: "Archived" no longer exists in schema
  - Counter-notification: "Terminal status 'Archived' no longer exists. Did you rename it to 'Closed'?"
  - T3 prompt to Admin to confirm and update profile

Fail condition: Agent continues treating "Archived" as terminal after it was renamed
```

---

## Assertions: Edge Cases

**ASSERT-EDGE-001: Deactivated user — nothing lost**
```
Input:
  USR-042 deactivated
  USR-042 had: 2 open HandoffCards, 3 owned risks, 1 in-progress assessment

Expected:
  - All 6 items accounted for in handoff notification
  - HandoffCards transferred to admin_equivalent role holders
  - Risk ownership: flagged as unassigned (not auto-reassigned)
  - In-progress assessment: SLA continues, admin notified to reassign assessor
  - AuditLog: user_deactivation_handoff with items_affected list

Fail condition: Any item disappears or becomes inaccessible after deactivation
```

**ASSERT-EDGE-002: Concurrent config save — both notified**
```
Input:
  Admin A saves Configuration Profile at 09:00:00
  Admin B saves Configuration Profile at 09:00:01 (concurrent write)

Expected:
  - Admin B's write is detected as concurrent
  - Both Admins notified: "Concurrent configuration change detected. Admin A's save: [summary]. Admin B's save: [summary]. Which should be applied?"
  - T3 gate: one Admin must confirm which version applies

Fail condition: Second write silently overwrites first without notification
```

**ASSERT-EDGE-003: CTRL-6 does not fire on terminal status risks**
```
Input:
  Control CTL-019 fails (Not Effective + Exceptions Noted)
  Linked risks: RISK-052 (active), RISK-044 (workflow_status="Archived")

Expected:
  - CTL-003 fires for RISK-052 only
  - RISK-044 skipped, logged
  - HandoffCard lists only RISK-052

Fail condition: RISK-044 included in CTRL-6 HandoffCard
```

---

## Eval Run Report Format

When running the eval suite, output in this format:

```
RISK AGENT EVAL RUN — [date]
Suite: [monitoring|breach-response|assessment|configuration|edge-cases|all]

PASS  ASSERT-MON-001  First assessment produces no breach alert
PASS  ASSERT-MON-002  Healthy subsequent assessment
FAIL  ASSERT-MON-005  Custom field monitoring — agent used OOTB field names instead of profile fields
WARN  ASSERT-BRE-003  Accept risk — review date defaulted to 1 year, expected prompt for custom date

SUMMARY
  Total: 20 assertions
  Pass:  18
  Fail:   1 (ASSERT-MON-005)
  Warn:   1 (ASSERT-BRE-003)

CRITICAL FAILURES (must fix before release):
  ASSERT-MON-005: Agent used hardcoded field name "residual_risk_score" instead of
  reading from Configuration Profile field_mapping. Profile had "net_risk_score".
  This would cause all monitoring to break for customised customers.

WARNINGS (investigate but not release-blocking):
  ASSERT-BRE-003: Review date defaulted without asking. Low risk but reduces governance transparency.
```

**Release criteria:** 0 FAILs. WARNs must be triaged — none that affect correctness may ship.
