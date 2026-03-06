# Flow Spec: [Org] - Prompt Dispatch

**Story:** 2.3 — Scheduled Prompt Dispatch Flow
**Trigger:** Scheduled recurrence
**Connections required:** SharePoint, Microsoft Teams
**Purpose:** Read all active SPOC records from InnovationTracker_Config and send each SPOC their PoC status prompt Adaptive Card via Teams DM. Update LastPromptedDate after successful delivery.

---

## Trigger

**Type:** Recurrence
**Frequency:** Daily (run every day; cadence filtering is done per-record in the loop)
**Suggested time:** 09:00 AM org local time
**Configuration:** Leave default timezone; set Start time to first desired run date

> Note: The flow runs daily but only sends cards to SPOCs whose `LastPromptedDate` + `CadenceDays` ≤ today. This means different SPOCs can have different cadences without multiple flows.

---

## Flow Variables (Initialize at top)

Declare these with the **Initialize variable** action before the main logic scope:

| Variable | Type | Initial value |
|---|---|---|
| `varRetryCount` | Integer | 0 |
| `varTodayDate` | String | `@{utcNow()}` (expression) |

---

## Main Logic Scope

Wrap everything below in a **Scope** action named `Main Logic`.

### Step 1 — Get active SPOC records

**Action:** SharePoint — Get items
**Site address:** `[ConfigListUrl]` (environment variable)
**List name:** `InnovationTracker_Config`
**Filter query:** `IsActive eq 1`
**Top count:** 500 (covers up to 50 orgs × 50 SPOCs)

Store result as: `SPOCItems` (implicit output of the Get items action)

---

### Step 2 — Loop over each SPOC record

**Action:** Control — Apply to each
**Input:** `value` output from Step 1 (Get items)

Inside the loop:

#### Step 2a — Check if prompt is due

**Action:** Control — Condition
**Condition expression:**

```
addDays(items('Apply_to_each')?['LastPromptedDate'], items('Apply_to_each')?['CadenceDays'])
```
is less than or equal to:
```
utcNow()
```

> If LastPromptedDate is blank (first-ever prompt), the comparison will fail — use the following expression to handle null:
> `if(empty(items('Apply_to_each')?['LastPromptedDate']), true, lessOrEquals(addDays(items('Apply_to_each')?['LastPromptedDate'], items('Apply_to_each')?['CadenceDays']), utcNow()))`

**If YES (prompt is due):** continue to Step 2b
**If NO (not yet due):** skip (end of branch — do nothing)

#### Step 2b — Set variables for this SPOC

**Action:** Set variable `varWorkItemId`
**Value:** `items('Apply_to_each')?['WorkItemId']`

**Action:** Set variable `varSpocUpn`
**Value:** `items('Apply_to_each')?['SpocUpn']`

**Action:** Set variable `varPocName`
**Value:** `items('Apply_to_each')?['PocName']`

**Action:** Set variable `varStage`
**Value:** `items('Apply_to_each')?['Stage']` (if you add a Stage column to config) OR hardcode based on which Feature is active — see Note below.

> **Note on Stage:** The Config list record maps to a specific ADO Feature work item ID (one Feature = one stage). The Stage value in the card (PoC / ProductFit / Scale) should be set from a `Stage` column you add to InnovationTracker_Config, populated when the record is created. This avoids an ADO lookup on every send.

**Action:** Set variable `varAdoOrgUrl`
**Value:** `items('Apply_to_each')?['AdoOrgUrl']`

**Action:** Set variable `varAdoProject`
**Value:** `items('Apply_to_each')?['AdoProject']`

**Action:** Set variable `varOrgName`
**Value:** `items('Apply_to_each')?['OrgName']`

#### Step 2c — Build the Adaptive Card body

**Action:** Compose
**Name:** `PromptCardBody`
**Inputs:** (paste the StatusPromptCard JSON from `cards/status-prompt-card.json`)

Within the JSON, replace the following placeholders using dynamic content expressions:

| Placeholder in JSON | Expression |
|---|---|
| `{{POC_NAME}}` | `variables('varPocName')` |
| `{{STAGE}}` | `variables('varStage')` |
| `{{WORK_ITEM_ID}}` | `string(variables('varWorkItemId'))` |
| `{{LAST_UPDATED}}` | `items('Apply_to_each')?['LastPromptedDate']` (or format with `formatDateTime(...)`) |

The stage criteria bullets are **hardcoded** in the card template per stage. Use a **Switch** action before Step 2c to select which card template variant to use based on `varStage`:

**Action:** Control — Switch
**On:** `variables('varStage')`

| Case | Value | Action |
|---|---|---|
| PoC stage | `PoC` | Compose with PoC-stage card criteria text |
| Product Fit stage | `ProductFit` | Compose with Product Fit criteria text |
| Scale stage | `Scale` | Compose with Scale criteria text |

Each Switch case outputs a `PromptCardBody` Compose with the appropriate criteria bullets embedded.

#### Step 2d — Send Adaptive Card to SPOC

**Action:** Microsoft Teams — Post an Adaptive Card to a Teams user and wait for a response

> **Important:** Use "Post an Adaptive Card to a Teams user and **wait for a response**" — this gives you the submission data back in the Response Handler. However, this blocks the loop until the SPOC responds or the card times out (up to 28 days Teams limit).

> **Recommended alternative approach:** Use **"Post adaptive card in a chat or channel"** (fire-and-forget) for the send, and handle responses in the separate Response Handler flow triggered by card submission. This is the preferred architecture — the Prompt Dispatch flow only sends; Response Handler only processes.

**Recommended: use "Post adaptive card in a chat or channel"**
**Recipient:** `variables('varSpocUpn')`
**Adaptive Card:** `outputs('PromptCardBody')` (the Compose output from Step 2c)
**Message:** (leave empty — card is the message)

#### Step 2e — Update LastPromptedDate in Config list

**Action:** SharePoint — Update item
**Site address:** `[ConfigListUrl]`
**List name:** `InnovationTracker_Config`
**Id:** `items('Apply_to_each')?['ID']`
**LastPromptedDate:** `utcNow()` formatted as date: `formatDateTime(utcNow(), 'yyyy-MM-dd')`

---

## Error Handling

**Add a parallel branch** to the `Main Logic` Scope. Configure it to run **only if** the Scope has failed.

Inside the failure branch:

### Step E1 — Increment retry counter
**Action:** Increment variable `varRetryCount` by 1

### Step E2 — Check retry count
**Action:** Control — Condition
**If `varRetryCount` is less than or equal to 3:**
→ Re-run the Main Logic scope (use **Control — Run after** configuration on a duplicate of the scope, or use a Do until loop wrapping the scope with `varRetryCount` as the exit condition)

**If `varRetryCount` is greater than 3:**
Continue to E3 and E4.

### Step E3 — Notify Mark via Teams
**Action:** Microsoft Teams — Post message in a chat or channel
**Post as:** Flow bot
**Post in:** Chat with Mark
**Recipient:** `[MarkUpn]` (environment variable)
**Message:**
```
⚠️ Innovation Tracker — Prompt Dispatch failed after 3 retries.
Flow: [Org] - Prompt Dispatch
Time: @{utcNow()}
Check the ErrorLog list for details.
```

### Step E4 — Log to ErrorLog list
**Action:** SharePoint — Create item
**Site address:** (ErrorLog site)
**List name:** `InnovationTracker_ErrorLog`
**Fields:**
- `FlowName`: `[Org] - Prompt Dispatch`
- `FailedAt`: `utcNow()`
- `ErrorMessage`: `@{body('Main_Logic')}` (scope error output)
- `WorkItemId`: `variables('varWorkItemId')` (last processed item)
- `OrgName`: `variables('varOrgName')`

---

## Stage Criteria Content

Hardcode the following criteria text in the appropriate Switch case card variants:

### PoC Stage criteria
```
• A working prototype or proof of concept exists
• Core technical feasibility has been demonstrated
• Initial user or stakeholder feedback collected
• Key risks and unknowns identified
• Go/no-go recommendation ready for Product Fit review
```

### Product Fit Stage criteria
```
• Problem-solution fit validated with target users
• Business case and ROI estimate defined
• Integration or dependency requirements assessed
• Resource and timeline estimate prepared
• Recommendation for Scale decision documented
```

### Scale Stage criteria
```
• Production readiness plan defined
• Organisational adoption and change management considered
• Metrics and success criteria agreed
• Handoff or sunset path documented
• Leadership sign-off obtained or in progress
```

> These criteria should be reviewed and adjusted by Mark/Michal before first deployment. They are the most important content in the card.

---

## Testing Checklist

- [ ] Flow runs on schedule without error
- [ ] Only records with `IsActive = true` receive cards
- [ ] Records with `LastPromptedDate + CadenceDays > today` are skipped
- [ ] Records with no `LastPromptedDate` (first run) receive a card
- [ ] Card appears in SPOC's Teams DM with correct PoC name, stage, and criteria
- [ ] Card contains the correct `workItemId` token (verify by inspecting submitted card data)
- [ ] `LastPromptedDate` is updated in Config list after card is sent
- [ ] If SharePoint Get items fails, Mark is notified and error is logged after 3 retries
- [ ] Card renders correctly on Teams mobile
