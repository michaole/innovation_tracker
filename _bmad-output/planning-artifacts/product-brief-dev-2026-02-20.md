---
stepsCompleted: [1, 2, 3, 4, 5, 6]
workflow_completed: true
inputDocuments:
  - brainstorming-session-2026-02-20.md
date: 2026-02-20
author: Michal
---

# Product Brief: dev

<!-- Content will be appended sequentially through collaborative workflow steps -->

## Executive Summary

The innovation funnel tracking system addresses a critical organizational pain point: PoC status updates depend entirely on manual discipline across disconnected systems (Confluence and ADO), creating a "status chaser" burden on the innovation team and eroding leadership confidence in the pipeline. The solution leverages the existing Microsoft ecosystem (ADO, Teams, Power BI, Power Automate) to create a friction-free status collection mechanism that brings updates to PoC owners rather than expecting them to seek out where and how to update. With a new Product Owner taking the lead, now is the ideal moment to establish a clean, sustainable process from day one.

---

## Core Vision

### Problem Statement

Innovation initiative (PoC) status tracking has no automated triggers — updates rely entirely on the innovation funnel Product Owner manually chasing PoC owners across the organization. PoC owners face confusion about where to update (Confluence vs ADO) and what exactly is expected, resulting in stale or missing status information across the pipeline.

### Problem Impact

- **Innovation team burden:** Solution Architects and the PO spend disproportionate time on administrative status collection instead of strategic innovation work
- **Leadership blind spots:** Senior stakeholders cannot see what is happening at what stage, leading to frustration and eroding confidence in the innovation pipeline
- **Stalled initiatives go unnoticed:** Without current status, PoCs can silently stall with no visibility or escalation
- **Wasted sync time:** Weekly meetings become status-collection sessions rather than strategic decision-making forums

### Why Existing Solutions Fall Short

The current approach uses Confluence for high-level summaries and ADO for detailed tracking, but this dual-system model creates two problems: update friction (people must go somewhere to update) and update confusion (people don't know which system matters or what's expected). There is no mechanism that prompts owners to update — the only trigger is a human chaser. This is unsustainable as the number of PoCs and teams grows.

### Proposed Solution

A zero-friction innovation tracking ecosystem built entirely on the existing Microsoft stack:
- **ADO** as the single source of truth for PoC status with clear stage-gate expectations
- **Teams bot** that proactively prompts PoC owners for quick status updates (bringing updates to them)
- **Power BI dashboard** providing real-time funnel visibility for leadership and the innovation team
- **Power Automate** as the integration glue connecting status changes to notifications, dashboards, and reporting

The system tells PoC owners exactly what is expected at each stage and makes responding take seconds, not minutes.

### Key Differentiators

- **Zero new tools** — built entirely within the Microsoft ecosystem teams already use daily
- **Pull-based updates** — the system comes to you via Teams, rather than requiring you to go somewhere
- **Clear expectations** — stage-specific criteria remove guesswork about what to report
- **Clean-slate timing** — new PO leadership enables process establishment without legacy baggage
- **Ambient visibility** — leadership gets always-current pipeline views without anyone preparing reports

---

## Target Users

### Primary Users

---

**Persona 1: Michal — Innovation Funnel Lead (Product Director)**

*"I need to know where everything stands without spending my day asking people."*

**Role & Context:** Accountable for the entire innovation pipeline — from idea intake through PoC outcomes to PI planning handoff. Regularly presents to senior leadership and steering committees. Owns the strategy; delegates day-to-day tracking to the PO.

**Problem Experience:** When status is stale, Michal is exposed — unable to answer leadership questions confidently, forced to scramble before presentations, or caught off-guard by PoCs that have silently stalled. The failure is visible at the worst moments.

**Success Vision:** Opens a dashboard before any meeting and instantly has an accurate, complete picture. Never has to ask anyone "what's the status?"

---

**Persona 2: Anna — Innovation Product Owner**

*"I spend too much time chasing people instead of managing the funnel."*

**Role & Context:** Day-to-day owner of the innovation funnel. Responsible for keeping ADO and Confluence current, coordinating with PoC owners across teams, and preparing status for leadership reviews. New to the role — starting fresh.

**Problem Experience:** The primary "chaser." Without automated triggers, every status update requires a manual follow-up. With 15–20 active PoCs across multiple teams, this is a near-constant overhead that crowds out higher-value work.

**Success Vision:** The system prompts PoC owners automatically. Anna reviews, escalates edge cases, and focuses on coordination and decision-making — not data collection.

---

**Persona 3: Tomasz — Solution Architect (Innovation Team)**

*"I need to know which PoCs need attention so I can help, not just audit."*

**Role & Context:** One of two SAs on the innovation team. Provides technical guidance to PoC teams. Needs accurate status to allocate his time where it matters most.

**Problem Experience:** Without current status, he can't distinguish PoCs that are genuinely progressing from ones that are stuck. He ends up either over-investing in healthy PoCs or missing the ones that need intervention.

**Success Vision:** A live view filtered by technical domain — instantly sees which PoCs are amber or red and where his expertise is needed.

---

**Persona 4: Piotr — PoC Owner (Developer/Architect from a product team)**

*"I'm heads-down on delivery — I know I should update status but I forget."*

**Role & Context:** Developer or architect from a product team, assigned as SPOC for one or two PoCs. Not part of the innovation team — the PoC is a side responsibility alongside his main product work.

**Problem Experience:** Doesn't object to providing status updates but has no natural trigger to do so. When chased, he complies — but the chase itself is an interruption. He's also sometimes unsure what level of detail is expected.

**Success Vision:** Gets a Teams message asking for a specific, structured update that takes under a minute. Knows exactly what's being asked. Done.

---

### Secondary Users

**The Lead Team — Senior Directors & VPs**

Not day-to-day users of the system, but key consumers of its output. They need pipeline visibility for:
- Monthly steering committee (scheduled, formal)
- Ad-hoc presentations and stakeholder questions

They expect accurate, visually clear status — either self-served from a Power BI dashboard or presented by Michal/Anna. They do not want to dig through ADO or Confluence. Their confidence in the innovation program is directly tied to how clearly status can be communicated.

---

### User Journey

**Piotr (PoC Owner) — Status Update Journey:**
1. **Trigger:** Teams bot sends a prompt on schedule — "Hi Piotr, quick update on [PoC Name]: Green / Amber / Red + one sentence. What's your status?"
2. **Response:** Piotr replies in 30 seconds directly in Teams
3. **Auto-sync:** Status written back to ADO automatically — no manual entry required
4. **Clarity:** Stage-specific criteria in ADO tell him exactly what's expected at his current stage

**Anna (Innovation PO) — Pipeline Management Journey:**
1. **Morning:** Opens Power BI dashboard — sees all 18 PoCs, their stages, and RAG status at a glance
2. **Intervention:** Filters for Amber/Red — identifies 2 PoCs needing follow-up, reaches out strategically
3. **Pre-meeting:** Dashboard is always current — no preparation needed for steering committee
4. **Escalation:** Time-in-stage alerts flag a PoC that's been stuck for 3 weeks — she escalates proactively

**Michal / Lead Team — Visibility Journey:**
1. **Ad-hoc:** VP asks "what's happening with the Databricks PoCs?" — Michal opens dashboard, answers in 60 seconds
2. **Monthly steering:** Dashboard serves as the presentation layer — no separate report preparation needed
3. **Strategic view:** Technology heatmap shows where innovation energy is concentrated vs. where there are gaps

---

## Success Metrics

The system succeeds when status collection becomes largely automatic, PoC owners respond promptly and completely, and leadership can self-serve pipeline visibility without asking the innovation team.

### Business Objectives

1. **Eliminate the chasing bottleneck** — The innovation PO's time spent on manual status collection drops from the primary overhead to a minor exception-handling activity
2. **Ensure always-current pipeline visibility** — Leadership can access accurate, up-to-date PoC status at any time without requesting a report or briefing
3. **Increase PoC owner accountability** — Status updates become a lightweight, expected behavior rather than an occasional manual effort
4. **Strengthen leadership confidence** — Monthly steering committee and ad-hoc requests are answered from the dashboard, not from memory or last week's Confluence page

### Key Performance Indicators

**For the Innovation Team (Anna / PO):**
- **Chasing rate:** % of PoCs requiring manual follow-up per week — target <20% at 3 months (down from ~100% today)
- **Status freshness:** % of PoCs with status updated within the last 7 days — target >90%
- **PO time on status collection:** Estimated hours per week spent chasing — target <1 hour/week at 3 months

**For PoC Owners:**
- **Bot response rate:** % of Teams bot prompts answered within 48 hours — target >80%
- **Update completeness:** % of responses that include all required fields (RAG + one sentence + blockers if amber/red) — target >90%
- **Time to respond:** Average time from bot prompt to reply — target <24 hours

**For Leadership:**
- **Ad-hoc status requests to innovation team:** Number of times leadership asks for status outside of scheduled reviews — target reduction of >70% at 3 months
- **Dashboard self-service rate:** Leadership accesses dashboard directly before/instead of asking — tracked qualitatively (survey or observation)
- **Steering committee prep time:** Time Anna/Michal spend preparing the monthly review — target <30 minutes (vs. current manual effort)

---

## MVP Scope

### Core Features (MVP — Phase 1)

**1. ADO as Single Source of Truth**
- Standardized PoC Feature template in ADO with required fields: PoC Name, SPOC, Technology Area, Funnel Stage, RAG Status, Last Update, Blockers
- Defined funnel stages: Idea → PoC In Progress → Results Ready → Product Fit → PI Planning Intake
- Stage-specific criteria embedded in ADO — each stage has 3–5 clear criteria defining what "done" means at that stage
- Migration of existing PoCs from Confluence into ADO structure

**2. Teams Bot for Automated Status Collection**
- Scheduled bot prompts sent directly to each PoC SPOC in Microsoft Teams
- Bot message includes the current stage criteria so the owner knows exactly what's expected
- Structured response format: RAG status (G/A/R) + one sentence update + blockers (if A/R)
- Bot writes responses directly back to ADO — no manual entry by SPOC or Anna
- Non-response escalation: flag to Anna after 48 hours without reply
- Scope: all PoC owners — internal teams and external/vendor teams using company Teams

### Out of Scope for MVP

| Feature | Phase |
|---|---|
| Power BI dashboard | Phase 2 |
| Power Automate integration glue | Phase 2 |
| Confluence auto-sync | Phase 2 |
| Innovation Intake Form (auto-creates ADO Feature) | Phase 3 |
| PoC Outcome Library | Phase 3 |
| Quarterly Innovation Demo Day | Phase 3 |
| Innovation Showcase Teams Channel | Phase 2 (quick win) |
| Time-in-stage alerts | Phase 2 |
| Technology heatmap view | Phase 3 |
| Monthly Innovation Scorecard | Phase 3 |
| Expansion to other organizations | Future |

### MVP Success Criteria

The MVP is considered successful and ready to scale when, within 3 months of launch:
- >80% of PoC owners respond to bot prompts within 48 hours without manual follow-up
- >90% of active PoCs have status updated within the last 7 days
- Anna spends <1 hour/week on status collection (vs. current baseline)
- No PoC silently stalls undetected for more than 2 weeks

### Future Vision

**Phase 2 — Portfolio Visibility (months 2–3):**
Power BI dashboard with funnel visualization, RAG status heatmap, stage drill-down, and time-in-stage alerts. Power Automate connects ADO to Teams notifications and Confluence display layer. Innovation Showcase Channel goes live.

**Phase 3 — Intelligence & Culture (months 4–6):**
- **Innovation Intake Form** — lightweight form that auto-creates a structured ADO Feature, so every PoC is "born tracked"
- **PoC Outcome Library** — completed PoCs archived with structured summaries, searchable by technology and domain, preventing duplicate work and capturing organizational learning
- **Quarterly Innovation Demo Day** — half-day event where mature PoCs present to product teams and leadership; natural forcing function for completion and PI planning handoff

**Future — Organization-Wide Expansion:**
The system scales beyond the initial innovation team to other departments and organizations within the company. The funnel model, ADO structure, and bot mechanism become a reusable blueprint for tracking innovation across the enterprise — establishing a shared innovation intelligence layer accessible to all technology teams.
