# Adaptive Card Templates

**Schema version:** 1.4
**Design system:** Microsoft Fluent (semantic tokens — inherited from Teams theme)
**Author tool:** [Adaptive Card Designer](https://adaptivecards.io/designer/)

---

## Templates

| File | Purpose | Used by |
|---|---|---|
| `status-prompt-card.json` | Weekly status prompt sent to PoC owners | Prompt Dispatch flow |
| `confirmation-state.json` | Shown after successful submission | Response Handler flow |
| `escalation-notification-card.json` | Escalation alert sent to Mark | Escalation flow |
| `error-state.json` | Shown when ADO write-back fails | Response Handler flow |

---

## Placeholder Reference

All placeholders use `{{UPPER_SNAKE_CASE}}` format. The Prompt Dispatch and Escalation flows replace these using Power Automate Compose actions or string interpolation before sending the card.

### status-prompt-card.json

| Placeholder | Source | Example |
|---|---|---|
| `{{POC_NAME}}` | InnovationTracker_Config.PocName | "AI-Assisted Triage PoC" |
| `{{STAGE}}` | InnovationTracker_Config.Stage | "PoC" / "ProductFit" / "Scale" |
| `{{LAST_PROMPTED_DATE}}` | InnovationTracker_Config.LastPromptedDate | "2026-02-27" |
| `{{STAGE_CRITERIA}}` | Hardcoded per stage — see below | "• Working prototype exists\n• ..." |
| `{{WORK_ITEM_ID}}` | InnovationTracker_Config.WorkItemId | "12345" |
| `{{SPOC_UPN}}` | InnovationTracker_Config.SpocUpn | "piotr@contoso.com" |

### confirmation-state.json

| Placeholder | Source |
|---|---|
| `{{POC_NAME}}` | From original card submission |
| `{{RAG_STATUS}}` | Submitted ragStatus value |
| `{{SUMMARY}}` | Submitted summary value |
| `{{TIMESTAMP}}` | utcNow() formatted |

### escalation-notification-card.json

| Placeholder | Source |
|---|---|
| `{{POC_NAME}}` | Trigger body pocName |
| `{{SPOC_UPN}}` | Trigger body spocUpn |
| `{{DAYS_OVERDUE}}` | Trigger body daysOverdue |
| `{{LAST_KNOWN_RAG}}` | ADO Custom.RAGStatus lookup |
| `{{LAST_KNOWN_SUMMARY}}` | ADO System.Description (first 200 chars) |
| `{{ADO_WORK_ITEM_URL}}` | Constructed URL to ADO work item |

---

## Stage Criteria Text

Use these exact strings for the `{{STAGE_CRITERIA}}` placeholder. Format with `\n• ` between bullets.

### PoC Stage
```
• A working prototype or proof of concept exists
• Core technical feasibility has been demonstrated
• Initial user or stakeholder feedback collected
• Key risks and unknowns identified
• Go/no-go recommendation ready for Product Fit review
```

### Product Fit Stage
```
• Problem-solution fit validated with target users
• Business case and ROI estimate defined
• Integration or dependency requirements assessed
• Resource and timeline estimate prepared
• Recommendation for Scale decision documented
```

### Scale Stage
```
• Production readiness plan defined
• Organisational adoption and change management considered
• Metrics and success criteria agreed
• Handoff or sunset path documented
• Leadership sign-off obtained or in progress
```

---

## Conditional Blockers Field (Action.ToggleVisibility)

The `blockers` Input.Text is hidden by default (`"isVisible": false`). The `Action.ToggleVisibility` button makes it visible when clicked.

**Important:** Adaptive Cards 1.4 does not support auto-showing the blockers field when a specific ChoiceSet value is selected — `Action.ToggleVisibility` must be triggered manually by the user clicking "Add blockers". This is acceptable for MVP.

If you require **automatic conditional reveal** (blockers field appears automatically on Amber/Red selection), you would need Adaptive Cards v1.5+ with `AssociatedInputs` or a custom Teams app. For MVP, manual toggle is the recommended approach.

---

## Testing in Adaptive Card Designer

1. Open https://adaptivecards.io/designer/
2. Paste the JSON from each template file
3. Replace `{{PLACEHOLDER}}` values with sample data manually
4. Preview in both Desktop and Mobile views
5. Verify tab order and keyboard navigation
6. Enable "High contrast" theme to verify accessibility

---

## Colour Token Reference (Fluent semantic)

| Token | Use case |
|---|---|
| `"style": "good"` | Container — success/confirmation states |
| `"style": "warning"` | Container — escalation/amber states |
| `"style": "attention"` | Container — error/blocked states |
| `"color": "Good"` | TextBlock — green status text |
| `"color": "Warning"` | TextBlock — amber status text |
| `"color": "Attention"` | TextBlock — red status / error text |
| `"style": "positive"` | Action.Submit — primary submit button |
| `"style": "destructive"` | Action.Submit — destructive actions only |

Never hardcode hex values. These tokens automatically adapt to Teams themes (default, dark, high contrast).
