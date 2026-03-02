---
stepsCompleted: [step-01-init, step-01b-continue, step-02-discovery, step-02b-vision, step-02c-executive-summary, step-03-success, step-04-journeys]
inputDocuments:
  - product-brief-dev-2026-02-20.md
  - research/domain-poc-to-product-funnel-research-2026-02-27.md
workflowType: 'prd'
project_name: dev
user_name: Michal
date: '2026-02-27'
classification:
  projectType: saas_b2b
  domain: general
  complexity: medium
  projectContext: greenfield
---

# Product Requirements Document - dev

**Author:** Michal
**Date:** 2026-02-27

## Executive Summary

The innovation funnel tracking system eliminates the manual status-chasing burden on the innovation team by replacing human-triggered update requests with an automated, pull-based mechanism. Built entirely on the existing Microsoft stack — Azure DevOps, Microsoft Teams, Power BI, and Power Automate — the system brings status collection to PoC owners rather than requiring them to seek out where and how to update. ADO serves as the single source of truth; a Teams bot handles structured prompting and auto-writes responses back to ADO with no manual entry required. The result: always-current pipeline visibility for leadership, near-zero administrative overhead for the innovation PO, and sub-60-second update effort for PoC owners.

**Target users:** Innovation Funnel Lead (Michal), Innovation PO (Mark), Solution Architects, PoC owners from product teams, and senior leadership consumers of pipeline visibility.

**North star:** >90% of innovation initiatives updated on time within 12 months of launch, with the system architecture proven and configurable for replication to a second organizational unit.

### What Makes This Special

The core insight is behavioral, not technical: the organization already has two tracking systems (ADO and Confluence) — what it lacks is a *trigger mechanism* that makes updating the natural, lowest-friction path. Every existing solution requires users to go somewhere to update; this system comes to them. The Teams bot delivers a structured, stage-specific prompt directly to each PoC owner and writes the response to ADO automatically — turning a multi-step administrative task into a 30-second reply. Zero new tooling is introduced; the system leverages infrastructure teams already use daily, which eliminates adoption friction entirely. The clean-slate timing of a new PO taking ownership enables process establishment without legacy baggage.

## Project Classification

- **Project Type:** Enterprise internal platform (SaaS B2B pattern) — multi-role, dashboard-driven, Teams-integrated
- **Domain:** Enterprise workflow automation (general; no regulatory compliance requirements)
- **Complexity:** Medium — multiple integration points (ADO REST API, Teams Bot Framework, Power Automate connectors), but established patterns and no novel technology
- **Project Context:** Greenfield — new system consuming and orchestrating existing enterprise infrastructure

## Success Criteria

### User Success

**PoC Owners (Piotr):**
- Responding to a bot status prompt takes under 60 seconds with no ADO or Confluence access required
- Stage-specific criteria in ADO make it unambiguous what information is expected at each funnel stage
- No PoC owner needs to remember to update — the system prompts them

**Innovation PO (Mark):**
- Time spent on manual status chasing drops from ~100% of PoCs requiring follow-up to <20% within 3 months of launch
- Pipeline state is visible at a glance without preparing a report or contacting anyone
- Amber/Red PoCs surface automatically — exception handling replaces routine chasing

**Innovation Lead & Leadership (Michal / Lead Team):**
- Any leadership status question is answerable within 60 seconds from the dashboard without involving Mark or Michal
- Monthly steering committee requires <30 minutes of preparation versus current manual effort

### Business Success

| Metric | Target | Timeframe |
|---|---|---|
| PoC owners responding to bot within 48 hours | >80% | 3 months |
| Active PoCs with status updated within 7 days | >90% | 3 months |
| PO time on manual status collection | <1 hour/week | 3 months |
| Reduction in ad-hoc leadership status requests | >70% | 3 months |
| Innovation initiatives updated on time (north star) | >90% | 12 months |
| Second org successfully onboarded to the system | 1 org | 12 months |

### Technical Success

- Teams bot delivers prompts reliably under normal operating conditions; downtime of hours to 1–2 days is acceptable — no formal SLA required
- ADO remains the single source of truth; bot responses write back to ADO automatically with no manual sync step
- System is configurable (funnel stages, prompt cadence, ADO field schema) without code changes to support onboarding of additional organizational units
- Onboarding documentation exists that enables a new org admin to configure and activate the system without engineering involvement

### Measurable Outcomes

- **Status freshness rate:** % of PoCs with ADO updated within 7 days — target >90% at 3 months
- **Bot response rate:** % of prompts answered within 48 hours — target >80%
- **Update completeness rate:** % of responses including all required fields (RAG + summary + blockers if A/R) — target >90%
- **Chasing rate:** % of PoCs requiring manual follow-up per sprint — target <20% at 3 months
- **Silent stall detection:** No PoC goes undetected as stalled for more than 2 weeks
- **Scale milestone:** At least one additional org unit onboarded within 12 months

## Product Scope

### MVP — Minimum Viable Product

**ADO as Single Source of Truth:**
- Standardized PoC Feature template with required fields: PoC Name, SPOC, Technology Area, Funnel Stage, RAG Status, Last Update, Blockers
- Defined funnel stages: PoC → Product Fit → Scale
- Stage-specific criteria (3–5 per stage) embedded in ADO defining what "done" means at each stage
- Migration of existing PoCs from Confluence into ADO structure

**Teams Bot for Automated Status Collection:**
- Scheduled prompts to each PoC SPOC in Microsoft Teams
- Prompt includes current stage criteria — owner knows exactly what's expected
- Structured response: RAG (G/A/R) + one-sentence update + blockers (if A/R)
- Auto-write back to ADO — no manual entry by SPOC or Mark
- Non-response escalation: flag to Mark after 48 hours without reply
- Coverage: all PoC owners on company Teams (internal and vendor)

**Onboarding & Documentation:**
- Configuration guide for funnel stages, prompt cadence, and ADO field schema
- Onboarding runbook enabling a new org admin to activate the system without engineering support

### Growth Features (Post-MVP — Phase 2)

- Power BI dashboard: funnel visualization, RAG heatmap, stage drill-down, time-in-stage alerts
- Power Automate: integration glue connecting ADO status changes to Teams notifications and Confluence display
- Innovation Showcase Teams Channel
- Time-in-stage automated alerts

### Vision (Future — Phase 3+)

- Innovation Intake Form: auto-creates structured ADO Feature for every new PoC ("born tracked")
- PoC Outcome Library: completed PoCs archived with structured summaries, searchable by technology and domain
- Quarterly Innovation Demo Day: forcing function for completion and PI planning handoff
- Enterprise-wide expansion: the funnel model, ADO structure, and bot mechanism become a reusable blueprint across the organization

## User Journeys

### Journey 1: Piotr (PoC Owner) — Status Update, Happy Path

Piotr is a developer on a product team, currently deep in a two-week sprint. The innovation PoC he's responsible for is a secondary responsibility — important, but not what he's thinking about on a Tuesday morning.

At 9:00am, a Teams message appears from the Innovation Bot: *"Hi Piotr — quick update on [PoC Name]. You're currently in the **PoC** phase. At this stage we need: (1) RAG status — Green/Amber/Red, (2) one sentence on where things stand, (3) any blockers if Amber or Red. Reply directly here."*

Piotr glances at the message. He's Green — things are moving. He types: *"Green. Integration with the data platform is 60% complete, on track for demo in 2 weeks."* He hits send and goes back to his sprint work. Elapsed time: 45 seconds.

The bot acknowledges receipt and writes the response directly to the ADO Feature for his PoC — RAG status updated, last-updated timestamp refreshed, summary recorded. Mark sees a fresh green entry in ADO. No chase required.

**Capabilities revealed:** Scheduled Teams bot prompting, phase-aware message content, free-text structured response parsing, ADO auto-write-back, response acknowledgement.

---

### Journey 2: Piotr (PoC Owner) — Non-Response Escalation

It's Wednesday. Piotr's PoC was prompted Monday morning. He saw it, meant to reply, and forgot. His ADO entry hasn't been updated in 9 days.

48 hours after the bot prompt, the system flags the non-response. Mark receives a Teams notification: *"No response received from Piotr for [PoC Name]. Last updated 9 days ago. Manual follow-up or override may be needed."*

Mark has two options: reach out to Piotr directly, or manually update ADO himself based on what he knows. He opens ADO, sets the RAG to Amber, adds a note — *"Awaiting SPOC response — PO follow-up initiated"* — and logs a blocker. The entry is current; leadership won't see a stale record.

Meanwhile, the bot sends Piotr a gentle follow-up in Teams. Piotr responds the next morning. ADO updates automatically, overwriting Mark's interim entry with Piotr's own words.

**Capabilities revealed:** 48-hour non-response detection, Mark notification via Teams, manual ADO override by PO, subsequent bot follow-up, response superseding manual entry.

---

### Journey 3: Mark (Innovation PO) — Daily Pipeline Management

Mark starts his Monday with ADO open. He has a filtered view showing all active PoCs sorted by RAG status and last-update date. In 90 seconds he has a complete picture: 14 Green, 3 Amber, 1 Red, 2 awaiting bot response.

He focuses on the Red. He clicks into the ADO Feature — the PoC owner flagged a dependency blocker on an external vendor. Mark forwards this to Christian (SA) for technical guidance and escalates to Michal for awareness. No meeting required; the context is all in ADO.

For the 2 awaiting response, he checks the timestamps — both were prompted yesterday. He'll wait another 24 hours before triggering a manual follow-up.

Before the weekly leadership sync, Mark opens ADO — the view is the same one he always uses. No report to prepare. He screenshots the RAG summary and pastes it into the Teams meeting chat. Done in 3 minutes.

**Capabilities revealed:** ADO filtered views by RAG/phase/freshness, manual PO override, ADO as the pre-meeting source of truth, no separate report generation needed.

---

### Journey 4: Mark (Innovation PO) — New PoC Onboarding & System Configuration

A product team approaches Michal with a new AI-assisted testing PoC they want to run. Michal approves it and tells Mark to get it tracked.

Mark opens ADO and creates a new Feature using the standardized PoC template. He fills in: PoC Name, SPOC (the team's lead developer, Karol), Technology Area (AI/ML), Phase (PoC), RAG (Green). He saves the Feature.

He then coordinates with the vendor to add Karol's Teams ID to the active SPOC list with the prompt cadence set to weekly. From next Monday, Karol will receive automated prompts without Mark doing anything further.

When it's time to onboard a second organizational unit, Mark uses the configuration runbook to guide their admin through the same setup — ADO project structure, field schema, bot SPOC list, cadence settings. No engineering involvement required.

**Capabilities revealed:** Manual ADO Feature creation with standardized template, SPOC roster management via vendor-maintained bot config, configurable prompt cadence, onboarding runbook for new orgs.

---

### Journey 5: Michal & Lead Team — Leadership Visibility

A VP stops Michal in the corridor: *"What's happening with the Databricks PoCs?"*

Michal pulls up ADO on his phone, filters by Technology Area: Databricks. Two PoCs — one Green in the Product Fit phase, one Amber still in PoC with a noted data access blocker. He answers the question on the spot, in under 60 seconds, with specifics.

At the monthly steering committee, Michal opens ADO rather than a prepared slide deck. He walks through the phase distribution — 6 active PoCs, 2 approaching Product Fit, 1 in Scale phase ready for PI planning handoff. The data is live; there is no "as of last Friday" caveat. Leadership asks follow-up questions; Michal drills in live. The meeting stays strategic.

**Capabilities revealed:** ADO technology-area filtering, mobile-accessible views, phase-based pipeline view, always-current data for ad-hoc and scheduled leadership interactions.

---

### Journey 6: Christian (Solution Architect) — Technical Domain Triage

Christian starts his week by checking which PoCs need technical attention. He filters ADO by Technology Area (his domains: cloud infrastructure, data engineering) and RAG status (Amber + Red). Three entries surface.

He reviews the blocker notes left by PoC owners in their bot responses. One is a genuine architectural dead-end — he schedules a working session with the team. One is a misunderstanding about platform limits — he replies in the ADO comments with a link to relevant documentation. One resolved itself since the last update.

He spends 20 minutes on targeted, high-value technical intervention instead of attending a status meeting to find out who needs help.

**Capabilities revealed:** ADO multi-field filtering (technology area + RAG), blocker notes visible in ADO, SA access to PoC detail without PO intermediary.

---

### Journey Requirements Summary

| Journey | Capabilities Required |
|---|---|
| Piotr — happy path | Scheduled bot prompts, phase-aware content, response parsing, ADO auto-write, acknowledgement |
| Piotr — non-response | 48hr detection, Mark notification, manual PO override, bot follow-up, response superseding override |
| Mark — pipeline management | ADO filtered views, manual override, no-prep leadership visibility |
| Mark — onboarding & config | ADO template, SPOC roster, configurable cadence, onboarding runbook |
| Michal / Leadership | ADO filtering by technology area + phase, mobile access, always-current data |
| Christian — SA triage | Multi-field ADO filtering, blocker visibility, direct ADO access |

