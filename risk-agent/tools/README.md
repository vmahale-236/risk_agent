# ERM MCP Server — Tool Reference

The ERM MCP Server exposes Risk Manager data as structured tools. All agents read and write through these tools. No direct database access.

---

## Core Tools

### list_object_types
List all available object types in this customer's Risk Manager configuration.
```
list_object_types()
→ [{ type_name, description, id_prefix }]
```

### get_object_schema
Get the full schema for an object type including all field names, types, and valid values. **Always call this before reading or writing to any object type.**
```
get_object_schema(type: string)
→ { type_name, fields: [{ name, type, required, options[] }] }
```

### get_object
Get a single object instance by type and ID.
```
get_object(type: string, id: string)
→ object record with all fields
```

### list_objects
List/filter instances of an object type.
```
list_objects(type: string, filters?: { field: value, ... }, limit?: number)
→ [object records]
```

### create_object
Create a new object record with validation.
```
create_object(type: string, fields: { field: value, ... })
→ created object record with generated id
```

### update_object
Update specific fields on an existing object.
```
update_object(type: string, id: string, fields: { field: value, ... })
→ updated object record
```

### list_relationships
Get all relationships for a given object — what it's linked to and what links to it.
```
list_relationships(object_type: string, object_id: string)
→ [{ relationship_type, related_object_type, related_object_id, related_object_name }]
```

### create_relationship
Create a link between two objects.
```
create_relationship(
  source_type: string, source_id: string,
  target_type: string, target_id: string,
  relationship_type: string
)
→ relationship record
```

### delete_relationship
Remove a link between two objects (not the objects themselves).
```
delete_relationship(relationship_id: string)
→ success confirmation
```

---

## Object Types Reference

| Prefix | Type | Description |
|--------|------|-------------|
| `RISK-` | risk | Enterprise risk records |
| `CTL-` | control | Internal controls |
| `REA-` | risk_event_assessment | Risk assessments linked to risks |
| `CA-` | control_assessment | Control assessments linked to controls |
| `ISS-` | issue | Issues, incidents, exceptions |
| `OU-` | org_unit | Organisational unit hierarchy |
| `USR-` | user | Platform users |
| `CAT-` | risk_category | Risk classification categories |
| `SM-` | scoring_matrix | Likelihood × Impact scoring definitions |
| `TAX-` | taxonomy_category | Risk/Control/Process taxonomy |
| `KRI-` | kri | Key Risk Indicators (future state) |
| `AL-` | audit_log_entry | Full audit trail of all actions |
| `WA-` | workflow_approval | Approval workflow records |
| `AR-` | agent_rule | Agent autonomy rules / constitution |
| `NL-` | notification_log | Record of all notifications sent |

---

## Relationship Types

| Type | Direction | Meaning |
|------|-----------|---------|
| `mitigates` | Control → Risk | This control mitigates this risk |
| `assesses` | REA → Risk | This assessment is for this risk |
| `assesses` | CA → Control | This assessment is for this control |
| `linked_to_issue` | Issue → Risk | This issue was raised from this risk |
| `linked_to_issue` | Issue → Control | This issue was raised from this control |
| `belongs_to` | Risk → OrgUnit | This risk belongs to this org unit |
| `belongs_to` | Control → OrgUnit | This control belongs to this org unit |
| `monitors` | KRI → Risk | This KRI monitors this risk |

---

## Agent Write Permissions Summary

| Agent | Can Read | Can Write | Cannot Write |
|-------|----------|-----------|-------------|
| Operations | All | risk, issue, relationship, audit_log, workflow_approval | scoring_matrix, constitution |
| Assessment | All | risk_event_assessment, control_assessment, risk (score fields only), audit_log | scoring_matrix, issue |
| Reporting | All | audit_log (publication events) | Everything else |
| Configuration | All | org_unit, risk_category, scoring_matrix, taxonomy_category, kri, workflow_approval, agent_rule, audit_log | risk (risk records), issue |

---

## Migration Path

| Stage | Backend | Change Required |
|-------|---------|-----------------|
| **Current** | ERM MCP Server (in development) | — |
| **Next** | DiligentMCP integration | MCP config only |
| **Production** | Production Risk Manager MCP | MCP config only |

Agent files require no changes when the MCP backend changes. Only the MCP server configuration changes.
