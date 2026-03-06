# Innovation Tracker — Power Automate Flow Specifications

**Generated:** 2026-03-06
**Status:** Ready for vendor implementation
**Target platform:** Power Automate (default environment) + Microsoft Teams + Azure DevOps

---

## Quick Reference

| Flow | File | Trigger | Story |
|---|---|---|---|
| [Org] - Prompt Dispatch | [01-prompt-dispatch.md](01-prompt-dispatch.md) | Scheduled (recurrence) | Story 2.3 |
| [Org] - Response Handler | [02-response-handler.md](02-response-handler.md) | Adaptive Card submission | Story 2.4 |
| [Org] - Non-Response Detection | [03-non-response-detection.md](03-non-response-detection.md) | Scheduled (recurrence) | Story 3.1 |
| [Org] - Escalation | [04-escalation.md](04-escalation.md) | HTTP (called by Non-Response Detection) | Story 3.2 |

## Adaptive Card Templates

| Template | File | Used by |
|---|---|---|
| StatusPromptCard | [cards/status-prompt-card.json](cards/status-prompt-card.json) | Prompt Dispatch |
| ConfirmationState | [cards/confirmation-state.json](cards/confirmation-state.json) | Response Handler (in-card update) |
| EscalationNotificationCard | [cards/escalation-notification-card.json](cards/escalation-notification-card.json) | Escalation |

---

## Prerequisites Before Building Any Flow

The following must exist in the target tenant before any flow is created:

### 1. ADO Custom Fields
These must be created in the ADO process template used by the Innovation Tracker project:

| Field reference | Display name | Type |
|---|---|---|
| `Custom.RAGStatus` | RAG Status | String |
| `Custom.Blockers` | Blockers | String (multi-line) |
| `Custom.LastUpdated` | Last Updated | DateTime |

### 2. ADO Area Path
- Create area path: `[Project]/Innovation Tracker`
- Assign this area path to all PoC Epic/Feature work items

### 3. Microsoft Lists
Two lists must exist in a SharePoint site accessible to the service account:

**InnovationTracker_Config** — one per org

| Column name | Type | Notes |
|---|---|---|
| `SpocUpn` | Single line of text | Teams UPN, e.g. piotr@org.com |
| `WorkItemId` | Number | ADO Feature work item ID |
| `PocName` | Single line of text | Human-readable PoC display name |
| `CadenceDays` | Number | Days between prompts, e.g. 7 |
| `AdoOrgUrl` | Single line of text | e.g. https://dev.azure.com/contoso |
| `AdoProject` | Single line of text | ADO project name |
| `OrgName` | Single line of text | Org prefix, e.g. Acme |
| `LastPromptedDate` | Date | Set by Prompt Dispatch; blank initially |
| `IsActive` | Yes/No | Default: Yes |

**InnovationTracker_ErrorLog** — shared

| Column name | Type |
|---|---|
| `FlowName` | Single line of text |
| `FailedAt` | Date and Time |
| `ErrorMessage` | Multiple lines of text |
| `WorkItemId` | Number |
| `OrgName` | Single line of text |

### 4. Power Automate Connections
Authenticate these connectors using the service account:
- **Azure DevOps** — Work Item read/write to Innovation Tracker area path only
- **Microsoft Teams** — Send messages as the service account/bot identity
- **SharePoint** — Read/write to the SharePoint site hosting both Lists

### 5. Teams App Manifest
Register via Teams Developer Portal:
- App name: `Innovation Tracker Bot`
- Identity: service account UPN
- Scope: Personal (1:1 DM with SPOCs)

---

## Solution Package

Export all 4 flows as a single managed solution named `InnovationTracker_v1.0`.

**Environment variables to parameterize per org:**

| Variable name | Purpose |
|---|---|
| `AdoOrgUrl` | ADO organization URL |
| `AdoProject` | ADO project name |
| `ConfigListUrl` | Full URL to InnovationTracker_Config SharePoint list |
| `ErrorLogListUrl` | Full URL to InnovationTracker_ErrorLog SharePoint list |
| `MarkUpn` | Innovation PO Teams UPN for escalation notifications |

---

## Implementation Sequence

1. Create Microsoft Lists schema (Config + ErrorLog)
2. Authenticate PA connections (ADO, Teams, SharePoint)
3. Build Adaptive Card JSON templates (test in Adaptive Card Designer)
4. Build Prompt Dispatch flow (Story 2.3)
5. Build Response Handler flow (Story 2.4)
6. Build Non-Response Detection flow (Story 3.1)
7. Build Escalation flow (Story 3.2)
8. End-to-end test in vendor tenant
9. Export as managed solution
10. Import to org tenant, configure environment variables, populate Config list
11. Register Teams app manifest, enable flows

---

## Cross-Flow Consistency Rules

All flows MUST use these exact identifiers — no variations:

### Adaptive Card field names
| Field | Exact name |
|---|---|
| Work item ID token | `workItemId` |
| RAG status | `ragStatus` |
| Status summary | `summary` |
| Blockers | `blockers` |
| Stage | `stage` |

### Flow variable naming
All flow-scoped variables use camelCase with `var` prefix:
`varWorkItemId`, `varSpocUpn`, `varRagStatus`, `varResponseReceived`, `varRetryCount`, `varPocName`, `varStage`, `varAdoOrgUrl`, `varAdoProject`

### Error handling — mandatory in every flow
1. Wrap main logic in a **Scope** action named `Main Logic`
2. Add parallel branch: configure to run **only on failure** of the scope
3. Failure branch: increment `varRetryCount`; if `varRetryCount` ≤ 3, retry main scope; if > 3, send Teams message to Mark AND create item in ErrorLog list
4. Never use ad-hoc condition checks for error paths — all errors go through the Scope failure branch

### Flow naming convention
`[OrgName] - [Capability]`
Examples: `Acme - Prompt Dispatch`, `Acme - Response Handler`
