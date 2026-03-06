# Flow Spec: [Org] - Escalation

**Story:** 3.2 — PO Escalation Notification & SPOC Follow-Up Prompt
**Trigger:** HTTP request (called by Non-Response Detection flow)
**Connections required:** Microsoft Teams, Azure DevOps (read-only for last known RAG), SharePoint
**Purpose:** When called by the Non-Response Detection flow, send Mark an escalation notification card in Teams (with last known RAG status + context), and send a follow-up prompt card to the overdue SPOC.

---

## Trigger

**Type:** When an HTTP request is received (HTTP trigger)

**Request body JSON schema:**

```json
{
  "type": "object",
  "properties": {
    "pocName": { "type": "string" },
    "spocUpn": { "type": "string" },
    "workItemId": { "type": "string" },
    "daysOverdue": { "type": "integer" },
    "lastPromptedDate": { "type": "string" },
    "orgName": { "type": "string" },
    "adoOrgUrl": { "type": "string" },
    "adoProject": { "type": "string" },
    "configItemId": { "type": "string" }
  }
}
```

> This flow is called once per overdue SPOC. Each call is independent.

---

## Flow Variables (Initialize at top)

| Variable | Type | Initial value |
|---|---|---|
| `varRetryCount` | Integer | 0 |
| `varPocName` | String | (empty) |
| `varSpocUpn` | String | (empty) |
| `varWorkItemId` | Integer | 0 |
| `varDaysOverdue` | Integer | 0 |
| `varLastKnownRag` | String | (empty) |
| `varLastKnownSummary` | String | (empty) |
| `varAdoWorkItemUrl` | String | (empty) |

---

## Main Logic Scope

Wrap all steps in a **Scope** action named `Main Logic`.

### Step 1 — Set variables from trigger body

**Action:** Set variable `varPocName`
**Value:** `triggerBody()?['pocName']`

**Action:** Set variable `varSpocUpn`
**Value:** `triggerBody()?['spocUpn']`

**Action:** Set variable `varWorkItemId`
**Value:** `int(triggerBody()?['workItemId'])`

**Action:** Set variable `varDaysOverdue`
**Value:** `triggerBody()?['daysOverdue']`

**Action:** Set variable `varAdoWorkItemUrl`
**Value:**
```
@{triggerBody()?['adoOrgUrl']}/@{triggerBody()?['adoProject']}/_workitems/edit/@{triggerBody()?['workItemId']}
```
This builds the direct ADO link for the "Open in ADO" button.

---

### Step 2 — Get last known RAG status from ADO

**Action:** Azure DevOps — Get a work item
**Organization:** derived from `triggerBody()?['adoOrgUrl']`
**Project:** `triggerBody()?['adoProject']`
**Work item ID:** `variables('varWorkItemId')`
**Fields:** `Custom.RAGStatus,System.Description,Custom.LastUpdated`

**Action:** Set variable `varLastKnownRag`
**Value:** `body('Get_a_work_item')?['fields']?['Custom.RAGStatus']`
(Use `if(empty(...), 'Not yet set', ...)` to handle blank)

**Action:** Set variable `varLastKnownSummary`
**Value:**
```
if(empty(body('Get_a_work_item')?['fields']?['System.Description']), 'No summary on record', substring(body('Get_a_work_item')?['fields']?['System.Description'], 0, 200))
```
Truncate to 200 characters to keep the card readable.

---

### Step 3 — Send escalation notification card to Mark

**Action:** Microsoft Teams — Post an adaptive card in a chat or channel
**Post as:** Flow bot
**Post in:** Chat
**Recipient:** `[MarkUpn]` (environment variable)
**Adaptive Card:** Use EscalationNotificationCard JSON from `cards/escalation-notification-card.json`

Replace placeholders in the card JSON:

| Placeholder | Expression |
|---|---|
| `{{POC_NAME}}` | `variables('varPocName')` |
| `{{SPOC_UPN}}` | `variables('varSpocUpn')` |
| `{{DAYS_OVERDUE}}` | `string(variables('varDaysOverdue'))` |
| `{{LAST_KNOWN_RAG}}` | `variables('varLastKnownRag')` |
| `{{LAST_KNOWN_SUMMARY}}` | `variables('varLastKnownSummary')` |
| `{{ADO_WORK_ITEM_URL}}` | `variables('varAdoWorkItemUrl')` |

**Override action in the card:** When Mark clicks "Override manually", the card's `Action.Submit` posts to the Response Handler flow with the `workItemId` token and Mark's input. This reuses the same Response Handler flow — no separate override endpoint needed.

---

### Step 4 — Send follow-up prompt card to overdue SPOC

**Action:** Microsoft Teams — Post an adaptive card in a chat or channel
**Post as:** Flow bot
**Post in:** Chat
**Recipient:** `variables('varSpocUpn')`
**Adaptive Card:** Use the same StatusPromptCard from `cards/status-prompt-card.json` with the same substitutions as Prompt Dispatch

Add a follow-up note above the criteria in the card by modifying the prompt copy TextBlock:

```
This is a follow-up — your previous update is overdue. Please take 60 seconds to fill this in.
```

Replace the standard prompt copy TextBlock text with the above for follow-up cards. You can do this with a Compose action that produces a modified card JSON variant.

---

### Step 5 — Return HTTP 200

**Action:** Response
**Status code:** 200
**Body:** `{"status": "ok"}`

---

## Error Handling

**Add a parallel branch** to `Main Logic`, configured to run **only on failure**.

### Step E1 — Increment retry counter
**Action:** Increment variable `varRetryCount` by 1

### Step E2 — Retry or give up
**Condition:** `varRetryCount` ≤ 3
- YES: retry `Main Logic` scope
- NO: continue

### Step E3 — Notify Mark of escalation failure
**Action:** Microsoft Teams — Post message in chat
**Recipient:** `[MarkUpn]`
**Message:**
```
⚠️ Innovation Tracker — Escalation flow failed after 3 retries.
PoC: @{variables('varPocName')}
SPOC: @{variables('varSpocUpn')}
Work Item: @{variables('varWorkItemId')}
Time: @{utcNow()}
Manual follow-up required.
```

### Step E4 — Log to ErrorLog
**Action:** SharePoint — Create item
**List:** `InnovationTracker_ErrorLog`
- `FlowName`: `[Org] - Escalation`
- `FailedAt`: `utcNow()`
- `ErrorMessage`: `@{body('Main_Logic')}`
- `WorkItemId`: `variables('varWorkItemId')`
- `OrgName`: `triggerBody()?['orgName']`

---

## Manual Override Flow (Mark clicks "Override manually")

When Mark clicks "Override manually" in the escalation card, the card's Action.Submit should:
1. Present a small inline form (or redirect to a second card) for Mark to enter RAG, Summary, and Blockers
2. Submit those values plus the `workItemId` to the Response Handler HTTP trigger URL

**Simplest implementation:** The "Override manually" button opens ADO in browser (`Action.OpenUrl` pointing to `varAdoWorkItemUrl`). Mark edits the ADO work item directly. This avoids building a second card and is fully supported by ADO.

**Alternative (in-card override form):** Add a second Adaptive Card variant — `OverrideCard` — that collects RAG + Summary from Mark and posts to Response Handler. More polished but more implementation work.

**MVP recommendation:** Use "Open in ADO" as the override path. Mark is familiar with ADO and can update fields directly. Relabel the button to "Override in ADO" for clarity.

---

## SPOC Response Superseding Mark's Override

After Mark manually overrides in ADO, if Piotr subsequently submits the status card:
- Response Handler receives the submission
- It overwrites `Custom.RAGStatus`, `Custom.Blockers`, `Custom.LastUpdated` with Piotr's values
- `System.Description` is appended (Mark's override entry is preserved as history, Piotr's words are added on top)

This is the correct behaviour — SPOC's own words supersede PO override. No special logic needed; Response Handler always overwrites.

---

## Testing Checklist

- [ ] Flow receives HTTP POST from Non-Response Detection and processes correctly
- [ ] Mark receives escalation card in Teams DM with correct PoC name, SPOC, days overdue
- [ ] Last known RAG and summary from ADO appear in Mark's notification card
- [ ] "Open in ADO" button in Mark's card navigates to the correct ADO work item URL
- [ ] Overdue SPOC receives follow-up prompt card in Teams DM
- [ ] Follow-up card contains "follow-up" language in prompt copy
- [ ] Follow-up card contains correct PoC name, stage, criteria, and `workItemId` token
- [ ] If ADO Get work item fails, escalation still proceeds with "Not yet set" as last known RAG
- [ ] Flow failure (after 3 retries) notifies Mark with PoC and SPOC details
- [ ] ErrorLog entry created on failure
- [ ] HTTP 200 returned on success
