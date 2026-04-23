# Risk Agent — Agentic ERM System

An agentic AI system for Enterprise Risk Management, built on the Claude Code agent SDK. Covers the full ERM lifecycle — from risk identification and assessment to monitoring, breach response, and board reporting.

---

## Architecture

```
CLAUDE.md                              ← Risk Maestro: routing, workflow, HITL, event bus
  ├── .claude/agents/                  ← Spawnable subagents (4 agents)
  ├── .claude/skills/                  ← Domain reasoning skills (4 skills)
  └── tools/                           ← ERM MCP Server (data access)
```

**CLAUDE.md** is the Risk Maestro orchestrator — routes tasks to agents, manages HITL touchpoints, and operates the inter-agent event bus. No agent calls another agent directly.

**Agents** know *what* to do and *when* — workflow steps, HITL touchpoints, handoff events, decision logic.

**Skills** know *how* to think — ERM domain rules, score comparison logic, notification IDs, HITL protocol.

**Tools** provide data access through the ERM MCP Server — 9 generic CRUD tools operating on 15 object types.

---

## Agent Hierarchy

| Agent | Phases | Status |
|-------|--------|--------|
| Configuration Agent | Pre-requisite setup, taxonomy, scoring, constitution | Implemented |
| Operations Agent | Identification, monitoring, breach, CTRL-6 | Implemented |
| Assessment Agent | Assessment lifecycle, SLA, aggregation, scoring | Implemented |
| Reporting Agent | Completeness, owner chasing, board report | Implemented |

---

## Skills

| Skill | What it provides |
|-------|----------------|
| `erm-domain-context` | Object model, user roles, pre-requisites, T1–T4 tiers, forbidden actions, schema-first rule |
| `risk-monitoring` | Score comparison, first vs. re-assessment detection, CTRL-6 failure matrix, mitigation loop |
| `human-interaction` | HITL types, plan-first protocol, HandoffCard format, role taxonomy |
| `notification-design` | All 50+ notification IDs, triggers, recipients, channels, suppression rules |

---

## Setup

### With Claude Code

```bash
cd risk-agent
claude
```

Claude Code auto-loads `CLAUDE.md`. The agent system activates immediately.

### With Cursor

The MCP Server is configured in `.cursor/mcp.json` (add your own after cloning).

Install MCP server dependencies:
```bash
cd tools/erm-mcp-server
npm install
```

---

## Running Agents in Isolation (Development)

During development, run individual agents directly without the orchestrator. This gives full real-time visibility of reasoning, tool calls, and decisions.

```
"Ignore the Risk Maestro role. Read .claude/agents/operations-agent.md and 
execute Job 3 (monitoring comparison) for RISK-052 which has:
  inherent_risk_score: Medium
  residual_risk_score: High
  scoring_scale: [Very Low, Low, Medium, High, Very High]"
```

CLAUDE.md is always loaded but explicit instructions take precedence.

---

## Key Business Rules (Summary)

1. **Pre-requisite gate** — Configuration Agent must confirm all 5 pre-requisites before Operations or Assessment Agents can write to the risk register.

2. **Score monitoring trigger** — Fires when BOTH `inherent_risk_score` AND `residual_risk_score` are non-null. First assessment only sets the baseline — no monitoring fires.

3. **Breach precedence** — If Operations Agent declares breach on a risk, Assessment Agent must pause immediately on that risk. Breach response cannot be blocked.

4. **Plan-first protocol** — Every T3 action presents a full plan (what/which objects/before-after/reversibility) before executing. No exceptions.

5. **Schema-first rule** — Always call `get_object_schema` before any read/write. Never hardcode field names. Customer schemas vary.

6. **No direct agent-to-agent calls** — All inter-agent communication routes through Risk Maestro event bus.

7. **Ordinal scoring** — Never compare scores as strings. Always use the ordinal ranking from the customer's configured scale.

---

## Inter-Agent Events

| Event | Sender → Receiver | When |
|-------|-------------------|------|
| `score_applied` | Assessment → Operations | Both scores populated on risk profile |
| `control_score_applied` | Assessment → Operations | Control assessment scores applied |
| `reassessment_requested` | Operations → Assessment | Human approves re-assessment post-mitigation or CTRL-6 |
| `ctrl6_chain_triggered` | Operations → Operations (audit) | Control failure detected |
| `breach_priority_lock` | Risk Maestro → Assessment | Pause assessment — breach takes precedence |
| `breach_priority_lock_released` | Risk Maestro → Assessment | Resume paused assessment |
| `reporting_summary_request` | Reporting → Operations + Assessment | Collect enriched summaries |
| `configuration_changed` | Configuration → All | Config applied — reload schemas |

---

## Source Documentation (Confluence)

All design decisions, workflows, and agent specifications are documented in Confluence (RCPPM space):

- Current ERM Flow — Risk Identification
- Current ERM Flow — Risk Assessment
- Current ERM Flow — Monitoring, Breach & Mitigation
- Current ERM Flow — Control Assessment & Reporting
- Operations Agent — AI Agent Definition
- Assessment Agent — AI Agent Definition
- Reporting Agent — AI Agent Definition
- Configuration Agent — AI Agent Definition
- Agent Overlay on Current ERM Flow
- Notification Design — When, Who, What, Channel
- Inter-Agent Handoff Contracts — Risk Maestro Communication Spec

---

## Migration Path

| Stage | MCP Backend | Change Required |
|-------|------------|-----------------|
| **Current** | ERM MCP Server (development) | — |
| **Next** | DiligentMCP | MCP config only — no agent changes |
| **Production** | Production Risk Manager MCP | MCP config only |
