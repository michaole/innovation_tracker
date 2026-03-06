# Flow Spec: [Org] - Response Handler

**Story:** 2.4 — Response Handling & ADO Write-Back Flow
**Trigger:** When an HTTP request is received (Adaptive Card submission webhook) OR "When a new response is submitted" (if using Teams Adaptive Card trigger)
**Connections required:** Microsoft Teams, Azure DevOps, SharePoint
**Purpose:** When a SPOC submits their status Adaptive Card, extract the response fields, write them to the correct ADO Feature work item, send an in-chat confirmation to the SPOC, and log any failures.

---

## Trigger

**Recommended trigger type:** HTTP request (When an HTTP request is received)

Power Automate Adaptive Card responses can be captured two ways:
1. **"Post an Adaptive Card to a Teams user and wait for a response"** in Prompt Dispatch (blocks the dispatch loop — not recommended for multiple SPOCs)
2. **HTTP webhook trigger** — the card's `Action.Submit` posts to a Flow HTTP trigger URL; Response Handler processes it independently

**Use option 2 (HTTP webhook).** This is the clean separation architecture: Prompt Dispatch sends, Response Handler receives.

**HTTP trigger configuration:**
- Method: POST
- Request body JSON schema (define so PA can parse it automatically):

```json
{
  "type": "object",
  "properties": {
    "workItemId": { "type": "string" },
    "ragStatus": { "type": "string" },
    "summary": { "type": "string" },
    "blockers": { "type": "string" },
    "stage": { "type": "string" },
    "spocUpn": { "type": "string" }
  }
}
```

> The `Action.Submit` in the Adaptive Card must include a `url` property pointing to this flow's HTTP trigger URL. Store the trigger URL as an environment variable `ResponseHandlerUrl` so it can be embedded in cards at send time by the Prompt Dispatch flow.

---

## Flow Variables (Initialize at top)

| Variable | Type | Initial value |
|---|---|---|
| `varRetryCount` | Integer | 0 |
| `varWorkItemId` | Integer | 0 |
| `varRagStatus` | String | (empty) |
| `varSummary` | String | (empty) |
| `varBlockers` | String | (empty) |
| `varStage` | String | (empty) |
| `varSpocUpn` | String | (empty) |
| `varTimestamp` | String | (empty) |

---

## Main Logic Scope

Wrap all steps below in a **Scope** action named `Main Logic`.

### Step 1 — Parse and set variables from trigger body

**Action:** Set variable `varWorkItemId`
**Value:** `int(triggerBody()?['workItemId'])`

**Action:** Set variable `varRagStatus`
**Value:** `triggerBody()?['ragStatus']`

**Action:** Set variable `varSummary`
**Value:** `triggerBody()?['summary']`

**Action:** Set variable `varBlockers`
**Value:** `if(empty(triggerBody()?['blockers']), '', triggerBody()?['blockers'])`

**Action:** Set variable `varStage`
**Value:** `triggerBody()?['stage']`

**Action:** Set variable `varSpocUpn`
**Value:** `triggerBody()?['spocUpn']`

**Action:** Set variable `varTimestamp`
**Value:** `formatDateTime(utcNow(), 'yyyy-MM-ddTHH:mm:ssZ')`

---

### Step 2 — Write to ADO Feature work item

**Action:** Azure DevOps — Update a work item

**Organization:** `[AdoOrgUrl]` (environment variable — extract org name from URL)
**Project:** `[AdoProject]` (environment variable)
**Work item ID:** `variables('varWorkItemId')`
**Type:** Feature

**Fields to update:**

| ADO field reference | Value expression | Notes |
|---|---|---|
| `Custom.RAGStatus` | `variables('varRagStatus')` | One of: Green, Amber, Red |
| `Custom.Blockers` | `variables('varBlockers')` | Empty string if Green |
| `Custom.LastUpdated` | `variables('varTimestamp')` | ISO 8601 datetime |
| `System.Description` | See below | Append mode |

**System.Description (append mode):**
ADO does not natively support field append in a single action. Use this approach:

1. **Action:** Azure DevOps — Get a work item
   - Work item ID: `variables('varWorkItemId')`
   - Store the result (current Description)

2. **Action:** Compose `AppendedDescription`
   ```
   @{body('Get_a_work_item')?['fields']?['System.Description']}

   ---
   **Status update — @{variables('varTimestamp')}**
   RAG: @{variables('varRagStatus')}
   Summary: @{variables('varSummary')}
   @{if(empty(variables('varBlockers')), '', concat('Blockers: ', variables('varBlockers')))}
   ```

3. **Update work item** sets `System.Description` to `outputs('AppendedDescription')`

---

### Step 3 — Send in-card confirmation to SPOC

**Action:** Microsoft Teams — Post a message in a chat or channel
**Post as:** Flow bot
**Post in:** Chat
**Recipient:** `variables('varSpocUpn')`

**Message body (use Adaptive Card format for consistency):**
Send a brief Teams message (not a full card) as the confirmation:

```
✅ Update recorded for your PoC.

RAG: @{variables('varRagStatus')}
Summary: @{variables('varSummary')}
Timestamp: @{variables('varTimestamp')}

Your update has been saved in ADO.
```

> **Preferred implementation:** Use a replacement Adaptive Card (ConfirmationState from `cards/confirmation-state.json`) if the Teams connector supports updating an existing card. If not, post the plain text confirmation message above.

---

### Step 4 — Return HTTP 200 response

**Action:** Response
**Status code:** 200
**Body:** `{"status": "ok"}`

This closes the HTTP trigger cleanly.

---

## Error Handling

**Add a parallel branch** to the `Main Logic` Scope, configured to run **only on failure**.

### Step E1 — Increment retry counter
**Action:** Increment variable `varRetryCount` by 1

### Step E2 — Check retry count
**Condition:** `varRetryCount` is less than or equal to 3
- YES: retry `Main Logic` scope
- NO: continue to E3 and E4

### Step E3 — Notify Mark via Teams
**Action:** Microsoft Teams — Post message in chat
**Recipient:** `[MarkUpn]` (environment variable)
**Message:**
```
⚠️ Innovation Tracker — Response write-back failed after 3 retries.
Flow: [Org] - Response Handler
Work Item ID: @{variables('varWorkItemId')}
SPOC: @{variables('varSpocUpn')}
Time: @{utcNow()}
Check the ErrorLog list for details.
```

Also send an error message to the SPOC so they know their submission didn't land:

**Action:** Microsoft Teams — Post message in chat
**Recipient:** `variables('varSpocUpn')`
**Message:**
```
⚠️ Your update could not be saved — please try again or contact Mark directly.
```

### Step E4 — Log to ErrorLog
**Action:** SharePoint — Create item
**List:** `InnovationTracker_ErrorLog`
- `FlowName`: `[Org] - Response Handler`
- `FailedAt`: `utcNow()`
- `ErrorMessage`: `@{body('Main_Logic')}`
- `WorkItemId`: `variables('varWorkItemId')`
- `OrgName`: (set from config or environment variable)

---

## ADO Field Mapping Reference

| Adaptive Card field | ADO field reference | ADO field type | Update mode |
|---|---|---|---|
| `ragStatus` | `Custom.RAGStatus` | String | Overwrite |
| `summary` | `System.Description` | HTML/string | Append with timestamp |
| `blockers` | `Custom.Blockers` | String | Overwrite |
| timestamp | `Custom.LastUpdated` | DateTime | Overwrite |

**RAG value mapping for ADO storage:**

| Card value | ADO stored value |
|---|---|
| `green` | `Green` |
| `amber` | `Amber` |
| `red` | `Red` |

> Use `toUpper(substring(variables('varRagStatus'),0,1))` + `substring(variables('varRagStatus'),1)` to capitalize if the card sends lowercase.

---

## Performance Target

- ADO write-back must complete within **2 minutes** of card submission
- If the flow takes longer, check ADO connector throttling in run history
- The HTTP trigger response (Step 4) should be sent immediately after ADO write succeeds to close the connection

---

## Testing Checklist

- [ ] Submitting a Green card writes `ragStatus=Green` to `Custom.RAGStatus` in ADO
- [ ] Submitting an Amber card writes `ragStatus=Amber` and the blockers text to `Custom.Blockers`
- [ ] `System.Description` in ADO contains the new update appended with timestamp (prior content preserved)
- [ ] `Custom.LastUpdated` is set to the submission timestamp
- [ ] SPOC receives Teams confirmation message after submission
- [ ] On ADO write failure: SPOC receives error message, Mark is notified, ErrorLog entry created
- [ ] On ADO write failure: retry occurs up to 3 times before notifying Mark
- [ ] HTTP trigger returns 200 on success
- [ ] Submitting via Teams Mobile works correctly
