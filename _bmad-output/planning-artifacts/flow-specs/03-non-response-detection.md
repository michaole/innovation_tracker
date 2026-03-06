# Flow Spec: [Org] - Non-Response Detection

**Story:** 3.1 — Non-Response Detection Flow
**Trigger:** Scheduled recurrence (runs daily)
**Connections required:** SharePoint, HTTP (to trigger Escalation flow)
**Purpose:** Daily scan of InnovationTracker_Config to find SPOC records where the prompt was sent but no response has been received within 48 hours of the configured cadence deadline. For each overdue record, trigger the Escalation flow.

---

## Trigger

**Type:** Recurrence
**Frequency:** Daily
**Suggested time:** 10:00 AM (run after Prompt Dispatch at 09:00 AM, allowing 1-hour gap)

---

## Flow Variables (Initialize at top)

| Variable | Type | Initial value |
|---|---|---|
| `varRetryCount` | Integer | 0 |
| `varTodayDate` | String | `@{utcNow()}` |

---

## How Overdue Detection Works

A record is considered overdue when:
- `IsActive` = true (SPOC is active)
- `LastPromptedDate` is not blank (at least one prompt has been sent)
- `LastPromptedDate` + `CadenceDays` + 2 (48-hour grace period) < today

The 48-hour grace period is added on top of the cadence to give SPOCs time to respond before escalating. For example, if `CadenceDays = 7` and the prompt was sent 9+ days ago with no response, escalation fires.

> **Note on "no response" detection:** This flow detects based on `LastPromptedDate` — the date the last prompt was sent. It does NOT check `Custom.LastUpdated` in ADO directly. This means: if a SPOC submits a response (which updates ADO via Response Handler), but `LastPromptedDate` is still old, the detection may still fire.
>
> **Recommended fix:** Add a `LastResponseDate` column to InnovationTracker_Config. The Response Handler flow should update this column when a successful ADO write-back occurs. Non-Response Detection then checks `LastResponseDate` instead of ADO.
>
> **Alternative (simpler):** Check `Custom.LastUpdated` in ADO directly via an ADO connector Get Work Item call inside the loop. More accurate but requires an ADO connection in this flow.
>
> **For MVP:** Use `LastPromptedDate` comparison only — the slight over-escalation (SPOC responded but date not yet synced) is tolerable and can be refined post-launch.

---

## Main Logic Scope

Wrap all steps in a **Scope** action named `Main Logic`.

### Step 1 — Get active SPOC records with past-due prompts

**Action:** SharePoint — Get items
**Site address:** `[ConfigListUrl]`
**List name:** `InnovationTracker_Config`
**Filter query:**
```
IsActive eq 1 and LastPromptedDate ne null
```
**Top count:** 500

> We filter for all active records with a LastPromptedDate, then apply the date arithmetic in the loop (SharePoint OData doesn't support date arithmetic in filter queries reliably).

---

### Step 2 — Loop over each record

**Action:** Control — Apply to each
**Input:** `value` from Step 1

Inside the loop:

#### Step 2a — Calculate deadline

**Action:** Compose `DeadlineDate`
**Expression:**
```
addDays(items('Apply_to_each')?['LastPromptedDate'], add(items('Apply_to_each')?['CadenceDays'], 2))
```

This adds `CadenceDays` + 2 (grace period days) to `LastPromptedDate`.

#### Step 2b — Check if overdue

**Action:** Control — Condition
**Condition:**
```
less(outputs('DeadlineDate'), utcNow())
```
(DeadlineDate is before now = overdue)

**If YES (overdue):** continue to Step 2c
**If NO (not overdue):** skip

#### Step 2c — Calculate days overdue

**Action:** Compose `DaysOverdue`
**Expression:**
```
div(sub(ticks(utcNow()), ticks(items('Apply_to_each')?['LastPromptedDate'])), 864000000000)
```

This calculates the number of days since the prompt was sent (integer division of ticks difference).

#### Step 2d — Trigger Escalation flow

**Action:** HTTP — HTTP
**Method:** POST
**URI:** `[EscalationFlowUrl]` (environment variable — the HTTP trigger URL of the Escalation flow)
**Headers:** `Content-Type: application/json`
**Body:**
```json
{
  "pocName": "@{items('Apply_to_each')?['PocName']}",
  "spocUpn": "@{items('Apply_to_each')?['SpocUpn']}",
  "workItemId": "@{items('Apply_to_each')?['WorkItemId']}",
  "daysOverdue": @{outputs('DaysOverdue')},
  "lastPromptedDate": "@{items('Apply_to_each')?['LastPromptedDate']}",
  "orgName": "@{items('Apply_to_each')?['OrgName']}",
  "adoOrgUrl": "@{items('Apply_to_each')?['AdoOrgUrl']}",
  "adoProject": "@{items('Apply_to_each')?['AdoProject']}",
  "configItemId": "@{items('Apply_to_each')?['ID']}"
}
```

---

## Error Handling

**Add a parallel branch** to the `Main Logic` Scope configured to run **only on failure**.

### Step E1 — Increment retry counter
**Action:** Increment variable `varRetryCount` by 1

### Step E2 — Retry or escalate to Mark

**Condition:** `varRetryCount` ≤ 3
- YES: retry `Main Logic` scope
- NO: continue

### Step E3 — Notify Mark
**Action:** Microsoft Teams — Post message in chat
**Recipient:** `[MarkUpn]`
**Message:**
```
⚠️ Innovation Tracker — Non-Response Detection failed after 3 retries.
Flow: [Org] - Non-Response Detection
Time: @{utcNow()}
Some SPOCs may not have received escalation prompts. Check ErrorLog.
```

### Step E4 — Log to ErrorLog
**Action:** SharePoint — Create item
**List:** `InnovationTracker_ErrorLog`
- `FlowName`: `[Org] - Non-Response Detection`
- `FailedAt`: `utcNow()`
- `ErrorMessage`: `@{body('Main_Logic')}`
- `WorkItemId`: 0 (not record-specific)
- `OrgName`: `[OrgName]` (environment variable)

---

## Dependency Note

This flow depends on:
- InnovationTracker_Config having accurate `LastPromptedDate` values (set by Prompt Dispatch)
- The Escalation flow being deployed and having a valid HTTP trigger URL (`EscalationFlowUrl`)

The Escalation flow URL changes each time it is deleted and recreated. If Escalation flow is rebuilt, update the `EscalationFlowUrl` environment variable.

---

## Testing Checklist

- [ ] Flow runs on schedule daily without error
- [ ] Records with `IsActive = false` are excluded
- [ ] Records with no `LastPromptedDate` are excluded
- [ ] Records where deadline has NOT passed are skipped (no escalation fired)
- [ ] Records where deadline HAS passed trigger the Escalation flow with correct payload
- [ ] `DaysOverdue` in the payload is a positive integer
- [ ] Multiple overdue records each trigger a separate Escalation flow call
- [ ] Flow failure notifies Mark and logs to ErrorLog after 3 retries
