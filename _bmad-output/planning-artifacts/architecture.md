---
stepsCompleted: [step-01-init, step-02-context, step-03-starter, step-04-decisions, step-05-patterns, step-06-structure]
inputDocuments:
  - prd.md
  - product-brief-dev-2026-02-20.md
  - research/domain-poc-to-product-funnel-research-2026-02-27.md
workflowType: 'architecture'
project_name: 'dev'
user_name: 'Michal'
date: '2026-03-03'
---

# Architecture Decision Document

_This document builds collaboratively through step-by-step discovery. Sections are appended as we work through each architectural decision together._

## Project Context Analysis

### Requirements Overview

**Functional Requirements:**
36 FRs across 6 areas: PoC data management (ADO Feature CRUD + stage definitions), automated status collection (bot scheduling + response parsing + ADO write-back), non-response detection and escalation, pipeline visibility (ADO filtered views), system configuration and administration (SPOC roster, stage config, multi-org onboarding), and access control (role-based ADO permissions + bot-only interaction for PoC owners).

The most architecturally significant FRs are:
- **FR8–FR13**: The full bot loop — schedule → prompt → parse → write-back → acknowledge
- **FR14–FR16**: Timer-based non-response detection and escalation
- **FR27–FR29**: Configuration-over-code for stages, cadence, and multi-org onboarding
- **FR11**: Free-text response parsing (highest technical uncertainty)

**Non-Functional Requirements:**
- Reliability: No formal SLA; downtime hours to 1–2 days tolerated; 3-retry write-back with PO notification on failure; ADO must remain queryable independent of bot availability
- Security: Minimum-permission service account; all data within Microsoft tenant; Azure AD SSO for human users
- Performance: Prompts delivered within 5 min of trigger; ADO write-back within 2 min of response; ADO views load within 5 sec for ≤200 records
- Scalability: ≤50 PoCs/org, ≤5 orgs — capacity bounded by Microsoft platform limits, not custom infrastructure

**Scale & Complexity:**
- Primary domain: Enterprise automation / bot integration
- Complexity level: Medium — integration-heavy, vendor-dependent, no novel technology
- Estimated architectural components: 5–6 (bot runtime, scheduler, response parser, ADO REST API client, configuration store, notification handler)

### Technical Constraints & Dependencies

- All components must run on Microsoft-managed infrastructure (Teams Bot Framework, ADO, Power Automate/Azure)
- Bot authenticates via a single Azure service account — no user-delegated permissions
- No data outside the Microsoft tenant
- Platform policy approval (Teams bot registration + service account ADO permissions) is the critical path dependency before production deployment
- Vendor builds and maintains the bot; architecture decisions must be implementable by an external vendor without internal engineering support

### Cross-Cutting Concerns Identified

- **Identity & authentication**: Service account for bot-to-ADO; Azure AD for human users; Teams AAD UPN used to match SPOC identity to ADO PoC record
- **Configuration management**: SPOC rosters, stage definitions, prompt cadence, and multi-org ADO project mapping must be updatable without code deployment
- **Error handling & observability**: 3-retry write-back failure → PO notification; needs structured logging for vendor support and debugging
- **Response parsing strategy**: Determines whether bot prompt uses free-text (requires NLP/rules) or adaptive cards (structured, no parsing needed)
- **Multi-org isolation**: Config must correctly route prompts and write-backs to the right ADO project per org

## Starter Template Evaluation

### Primary Technology Domain

Microsoft Power Platform automation — no custom bot code. The system is implemented as a set of Power Automate flows using native connectors, with Teams Adaptive Cards replacing free-text conversation.

### Evaluated Options

**Bot Framework SDK v4**: Eliminated — support ended December 31, 2025; project archived. Not suitable for new development.

**Teams SDK v2 (TypeScript)**: Viable but over-engineered for this use case. Requires Azure hosting, custom code maintenance, and TypeScript expertise from the vendor. Adds complexity without proportionate benefit at this scale.

**Power Automate + Teams Adaptive Cards** *(selected)*: Native Microsoft low-code platform. Scheduled flows, ADO connector, and Adaptive Cards cover all MVP requirements without custom code.

### Selected Approach: Power Automate + Teams Adaptive Cards

**Rationale:**
- Adaptive Cards deliver a structured form (RAG dropdown + text fields) to SPOCs — eliminating free-text response parsing entirely (resolves FR11's highest technical risk)
- Power Automate native ADO connector handles write-back without a custom API client
- Scheduled flows handle prompt dispatch and 48-hour non-response detection natively
- No hosting infrastructure to provision or maintain
- Configuration-over-code requirement (FR27–29) is naturally satisfied — flows are configurable via UI without code deployment
- Aligns with Power Automate already planned for Phase 2 — MVP and Phase 2 share the same platform
- Vendor can implement and maintain without deep engineering expertise

**Platform initialization:**
- No CLI scaffold; implementation begins with Power Automate environment setup in the existing Microsoft 365 tenant
- Teams app manifest created via Teams Developer Portal to register the Adaptive Card sender identity
- ADO connector authenticated via the designated Azure service account

**Architectural components provided by this approach:**

**Scheduling:** Power Automate Recurrence trigger — configurable cadence per flow or per SPOC group; no custom scheduler needed

**Prompt delivery:** Teams connector in Power Automate — sends Adaptive Cards directly to SPOC via Teams chat; no bot registration required beyond app manifest

**Response collection:** Adaptive Card structured response — RAG status (dropdown: Green/Amber/Red), summary (text field), blockers (conditional text field); responses returned directly to Power Automate flow as structured JSON; no parsing logic needed

**ADO write-back:** Azure DevOps connector in Power Automate — native connector updates Work Item fields (RAG, summary, blockers, last-updated timestamp) without custom REST API client

**Non-response detection:** Power Automate delay + condition logic — flow waits 48 hours after prompt; if no response token received, branches to escalation path

**Escalation notifications:** Teams connector — sends notification message to Mark's Teams chat with PoC name and days-since-update

**Configuration store:** SharePoint List or Excel table in SharePoint — stores SPOC roster (Teams UPN, ADO work item ID, PoC name, prompt cadence, org/project mapping); updated by Mark without touching flows

**Note:** Implementation begins with Power Automate environment provisioning and ADO connector authentication using the designated service account. This is the first implementation story.

## Core Architectural Decisions

### Decision Priority Analysis

**Critical Decisions (Block Implementation):**
- ADO Work Item hierarchy (Epic/Feature structure)
- Service account OAuth for PA-to-ADO connector
- Response token strategy (work item ID embedded in Adaptive Card)
- Managed solution packaging for vendor handoff

**Important Decisions (Shape Architecture):**
- One flow per capability (modular flow design)
- Separate flow set per org (multi-org isolation)
- Microsoft Lists as sole configuration store
- Hardcoded stage criteria in Adaptive Card template

**Deferred Decisions (Post-MVP):**
- Advanced telemetry / Application Insights (revisit if orgs > 5)
- Dedicated PA environment (revisit if governance requirements increase)

### Data Architecture

**ADO Work Item Structure:**
- Epic = PoC initiative entry point (one per PoC)
- Feature = stage (3 Features per Epic: PoC / Product Fit / Scale)
- Rationale: Maps naturally to the funnel stages; ADO hierarchical views work cleanly at Epic level

**Stage Criteria Storage:**
- Hardcoded in Adaptive Card template
- Rationale: Stages are stable and defined by process design, not runtime config; eliminates a config lookup on every prompt

**Configuration Store:**
- Microsoft Lists (SharePoint-backed)
- Stores: SPOC roster (Teams UPN, ADO work item ID, PoC name, prompt cadence, org/project mapping)
- Updated by Mark directly via browser — no flow redeployment needed

### Authentication & Security

**Bot-to-ADO Authentication:**
- Service account OAuth — single Azure AD user account authenticates the Power Automate ADO connector
- Minimum-permission scope: Work Item read/write only
- Credential stored encrypted in PA connection object; no Azure Key Vault required at this scale

**Human User Access:**
- Microsoft 365 native permissions on Microsoft Lists
- Mark (admin) has edit access; no custom RBAC layer

### Flow Architecture & Communication

**Flow Granularity:**
- One flow per capability:
  1. Prompt dispatch flow (scheduled — sends Adaptive Cards to SPOCs)
  2. Response handling flow (triggered on card submission — writes back to ADO)
  3. Non-response detection flow (scheduled — checks for overdue responses)
  4. Escalation flow (triggered by non-response detection — notifies Mark)

**Response Routing:**
- ADO work item ID embedded as a data token in the Adaptive Card at send time
- Token returned with card submission response — flow uses it directly for ADO write-back
- No secondary lookup against config list required

**Multi-Org Routing:**
- Separate cloned flow set per org
- Each flow set parameterized with org-specific ADO URL, project name, and config list reference
- Rationale: Keeps per-org logic isolated; failure in one org does not affect others; simpler to onboard/offboard an org

### Infrastructure & Deployment

**Power Automate Environment:**
- Default environment (sufficient for this scale and team)

**Solution Packaging:**
- Managed solution used for vendor handoff
- Vendor develops and tests in their own tenant, exports as managed solution, imports to org tenant
- Enables clean deployment, version control, and rollback

**Monitoring & Alerting:**
- Power Automate built-in run history for debugging
- Email notification on flow failure (built-in PA mechanism)
- Mark notified on write-back failures per NFR (3-retry → PO notification handled by escalation flow)

**Environment Configuration:**
- All environment-specific values (ADO org URL, project name, service account) stored in Microsoft Lists
- No .env files, Azure App Config, or Key Vault required

### Decision Impact Analysis

**Implementation Sequence:**
1. Microsoft Lists config table setup (prerequisite for all flows)
2. PA-to-ADO connector authentication (service account)
3. Adaptive Card template design (RAG dropdown, summary, blockers + work item ID token)
4. Prompt dispatch flow
5. Response handling flow (ADO write-back)
6. Non-response detection flow
7. Escalation flow
8. Managed solution export and import to org tenant
9. Teams app manifest registration

**Cross-Component Dependencies:**
- Config list must exist before any flow can run
- Adaptive Card template token design gates response handling flow
- Service account ADO permissions must be provisioned before connector auth
- Managed solution packaging happens after all flows are tested in vendor tenant

## Implementation Patterns & Consistency Rules

### Critical Conflict Points Identified
6 areas where AI agents implementing different flows could make incompatible choices:
flow naming, Adaptive Card field schema, Microsoft Lists column naming, flow variable naming,
error handling structure, and ADO field mapping.

### Flow Naming Pattern
Convention: `[OrgName] - [Capability]`
Examples:
- `Acme - Prompt Dispatch`
- `Acme - Response Handler`
- `Acme - Non-Response Detection`
- `Acme - Escalation`

When cloning for a new org, replace only the org prefix.

### Adaptive Card Field Schema
All flows must use these exact JSON field names when sending and reading Adaptive Card responses:

| Field | Name | Type | Values |
|---|---|---|---|
| Work item ID token | `workItemId` | string | ADO work item ID |
| RAG status | `ragStatus` | string | `Green`, `Amber`, `Red` |
| Summary | `summary` | string | free text |
| Blockers | `blockers` | string | free text, optional |
| Stage | `stage` | string | `PoC`, `ProductFit`, `Scale` |

Anti-pattern: Do NOT use `rag_status`, `RAGStatus`, `work_item_id` — field name mismatches break the response handling flow.

### Microsoft Lists Schema
Config list column names (exact — do not vary):

| Purpose | Column Name | Type |
|---|---|---|
| SPOC Teams UPN | `SpocUpn` | Single line of text |
| ADO Work Item ID | `WorkItemId` | Number |
| PoC Name | `PocName` | Single line of text |
| Prompt cadence (days) | `CadenceDays` | Number |
| ADO Org URL | `AdoOrgUrl` | Single line of text |
| ADO Project Name | `AdoProject` | Single line of text |
| Last prompted date | `LastPromptedDate` | Date |
| Org identifier | `OrgName` | Single line of text |
| Active flag | `IsActive` | Yes/No |

### Flow Variable Naming
All flow-scoped variables use camelCase with `var` prefix:
- `varWorkItemId`, `varSpocUpn`, `varRagStatus`, `varResponseReceived`, `varRetryCount`

### Error Handling Pattern
All flows MUST implement:
1. Wrap main logic in a **Scope** action named `Main Logic`
2. Add parallel branch configured to run on failure
3. Failure branch: increment `varRetryCount`; if ≤3 retry main scope; if >3 send email to Mark and log to `ErrorLog` list
4. `ErrorLog` list columns: `FlowName`, `FailedAt`, `ErrorMessage`, `WorkItemId`, `OrgName`

Anti-pattern: Do NOT use ad-hoc condition checks for error handling — all error paths must go through the Scope failure branch.

### ADO Work Item Field Mapping
Exact ADO field references for the response handling flow:

| Adaptive Card field | ADO REST field reference |
|---|---|
| `ragStatus` | `Custom.RAGStatus` |
| `summary` | `System.Description` (append mode) |
| `blockers` | `Custom.Blockers` |
| timestamp | `Custom.LastUpdated` |

Note: Custom ADO fields (`Custom.RAGStatus`, `Custom.Blockers`, `Custom.LastUpdated`) must be created in ADO before flows can write to them. This is a prerequisite story.

### Enforcement Guidelines

**All AI Agents MUST:**
- Use exact field names from the Adaptive Card schema — no variations
- Use exact column names from the Microsoft Lists schema — no variations
- Implement the Scope-based error handling pattern in every flow
- Prefix flow names with the org identifier
- Use camelCase with `var` prefix for all flow variables

**Pattern Verification:**
- Before handoff, vendor must provide a flow inventory checklist confirming naming compliance
- ADO custom fields must be verified to exist before solution import

## Project Structure & Boundaries

### Solution Package Structure

The deliverable is a Power Automate managed solution named `InnovationTracker`.
Vendor develops in their own tenant, exports, and imports to the org tenant.

```
InnovationTracker (Managed Solution)
├── Flows/
│   ├── [Org] - Prompt Dispatch          # Scheduled; reads config list, sends Adaptive Cards
│   ├── [Org] - Response Handler         # Triggered on card submission; writes back to ADO
│   ├── [Org] - Non-Response Detection   # Scheduled; checks LastPromptedDate vs threshold
│   └── [Org] - Escalation               # Triggered by Non-Response Detection; notifies Mark
├── Connections/
│   ├── ADO Connector                    # Authenticated via service account OAuth
│   ├── Teams Connector                  # Sends Adaptive Cards + notifications
│   └── SharePoint Connector             # Reads/writes Microsoft Lists
└── Environment Variables/
    ├── AdoOrgUrl                        # Per-org ADO organization URL
    ├── AdoProject                       # Per-org ADO project name
    └── ConfigListUrl                    # Per-org Microsoft Lists URL
```

One solution package per org. Cloned and re-parameterized at onboarding.

### Microsoft Lists Schema

**List: `InnovationTracker_Config`** (one per org)

| Column | Type | Purpose |
|---|---|---|
| `SpocUpn` | Single line | SPOC Teams UPN for card delivery |
| `WorkItemId` | Number | ADO Feature work item ID |
| `PocName` | Single line | Human-readable PoC name |
| `CadenceDays` | Number | Prompt frequency in days |
| `AdoOrgUrl` | Single line | ADO org URL for this record |
| `AdoProject` | Single line | ADO project name for this record |
| `OrgName` | Single line | Org identifier (matches flow prefix) |
| `LastPromptedDate` | Date | Set by Prompt Dispatch flow on send |
| `IsActive` | Yes/No | Exclude inactive SPOCs from prompts |

**List: `InnovationTracker_ErrorLog`** (shared)

| Column | Type | Purpose |
|---|---|---|
| `FlowName` | Single line | Which flow failed |
| `FailedAt` | Date/Time | Failure timestamp |
| `ErrorMessage` | Multi-line | PA error detail |
| `WorkItemId` | Number | Work item being processed |
| `OrgName` | Single line | Org context |

### ADO Configuration

**Work Item Hierarchy:**
```
Epic: [PoC Initiative Name]
├── Feature: PoC Stage
├── Feature: Product Fit Stage
└── Feature: Scale Stage
```

**Custom Fields (must be created before solution import):**

| Field Reference | Type | Purpose |
|---|---|---|
| `Custom.RAGStatus` | String | RAG status from Adaptive Card |
| `Custom.Blockers` | String | Blockers text from Adaptive Card |
| `Custom.LastUpdated` | DateTime | Timestamp of last SPOC response |

**Area Path:** `[Project]/Innovation Tracker`

### Teams App Manifest

Registered via Teams Developer Portal to give the Adaptive Card sender a recognizable Teams identity.

| Setting | Value |
|---|---|
| App name | `Innovation Tracker Bot` |
| Identity | Service account UPN |
| Scope | Personal (1:1 chat with SPOCs) |

### Requirements to Structure Mapping

| FR Capability Area | Implementation Location |
|---|---|
| FR1–FR7: PoC data management (Epic/Feature CRUD) | ADO work item hierarchy + Config list |
| FR8–FR13: Bot loop (schedule → card → write-back) | Prompt Dispatch + Response Handler flows |
| FR14–FR16: Non-response detection & escalation | Non-Response Detection + Escalation flows |
| FR17–FR22: Pipeline visibility (ADO views) | ADO filtered queries / boards (no flow needed) |
| FR23–FR29: Config & administration | Microsoft Lists `InnovationTracker_Config` |
| FR30–FR36: Access control | ADO permissions + M365 List permissions |

### Integration Boundaries

**Prompt Dispatch Flow → Teams:**
- Reads `InnovationTracker_Config` (active SPOCs)
- Sends Adaptive Card via Teams connector to `SpocUpn`
- Updates `LastPromptedDate` in config list
- Boundary: Teams delivery is fire-and-forget; no read-receipt confirmation

**Response Handler Flow ← Teams → ADO:**
- Triggered by Adaptive Card submission
- Reads `workItemId` token from card response
- Writes `ragStatus`, `summary`, `blockers`, `Custom.LastUpdated` to ADO Feature
- Boundary: ADO write is the terminal action; no callback to SPOC needed

**Non-Response Detection Flow → Escalation Flow:**
- Scheduled; queries config list for records where `LastPromptedDate` + `CadenceDays` < today
- Passes PoC name, SPOC UPN, and days overdue to Escalation flow via flow run trigger
- Boundary: Escalation is a separate flow — decoupled for independent failure handling

**Data Flow:**
```
Config List → Prompt Dispatch → Teams (Adaptive Card)
                                      ↓
                               SPOC Response
                                      ↓
                          Response Handler → ADO Feature
                                      ↓
                Non-Response Detection → Escalation → Mark (Teams)
```

### Development & Deployment Workflow

**Vendor Development Tenant:**
1. Create Microsoft Lists schema (Config + ErrorLog)
2. Create connections (ADO, Teams, SharePoint)
3. Build and test all 4 flows
4. Export as managed solution `InnovationTracker_v1.0`

**Org Tenant Import:**
1. Provision service account + ADO permissions
2. Create ADO custom fields
3. Create Microsoft Lists (Config + ErrorLog)
4. Import managed solution
5. Configure environment variables (AdoOrgUrl, AdoProject, ConfigListUrl)
6. Populate config list with SPOC roster
7. Register Teams app manifest
8. Enable flows
