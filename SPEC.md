# Advisor Chat Console – Prosperr

**Owner:** Product Team
**Status:** Draft for engineering review
**Priority:** High
**Scope of this document:** Chat workflow only, structured by persona.

## Personas at a Glance

| Persona | Core Problem | Phase 1 Outcome |
|---|---|---|
| **Tax Advisor** | Cannot tell which chat needs them first, and a reply is treated as a finish. | Chats ranked by risk; a reply no longer clears ownership; closure is an explicit act. |
| **Admin** | Finds out about at-risk chats from the client, not the system. | Two dashboard widgets showing chat load and which agents carry risk. Read-only. |

---
---

# Part 1 — Tax Advisor

## 1.1 Problem

| Item | Description |
|---|---|
| **Target User** | Prosperr tax advisor working a mixed book — ITR filing, advance tax, notices, planning. |
| **Pain Point** | 49% of client chats received zero advisor reply, ever. Of those answered, almost none are ever closed. |
| **User Context** | Advisors triage by scrolling a raw inbox. Nothing indicates which chat is closest to breaching, or which they have already touched. |
| **Reason** | SLA is measured on **first response only**, so a holding reply ("ok, will check") fully satisfies it. There is no closure signal, so an answered chat is indistinguishable from a finished one. |
| **Consequence** | Clients wait, filing deadlines slip, and work stalls invisibly. Open chats grew ~47% Jun→Jul while volume grew only ~17%. |

## 1.2 Hypothesis

| Item | Description |
|---|---|
| **Expected Behaviour** | If status is **derived** from message and action state rather than self-declared, and chats are ranked by aging + time since last action + SLA proximity, the advisor opens the right chat first and closes what they finish. |
| **Reason** | Advisors are rewarded for replying and never asked to finish. Making a reply insufficient to clear ownership targets that directly. Refer **Annexure A6**. |

## 1.3 Scope

| Item | Details |
|---|---|
| **In Scope** | • Derived status: New chats / Open sessions / Action pending / Closed<br>• Two SLA clocks — first response 12 business hours, resolution 48 hours<br>• Priority scoring and ranked chat list<br>• Action-pending flag and holding-reply detection<br>• Closure with disposition capture<br>• Reopen behaviour on closed chats<br>• Primary / Secondary ownership facet |
| **Out of Scope** | • Task creation and task-gated closure — *Phase 2*<br>• Document request — *Phase 2*<br>• Meeting / MOM handling — *Phase 2*<br>• Per-topic granularity inside a single chat<br>• Client-facing status tracker |

## 1.4 Acceptance Criteria

| Feature | Acceptance Criteria |
|---|---|
| **Chat Status** | As a system, I should derive every chat's status from who sent the last message plus the action-pending flag. No status dropdown exists anywhere in the UI. Refer **Annexure A1**. |
| **First Response SLA** | As a system, I should measure first response against a 12-hour target counted in business hours only — 09:00 to 20:00, Saturdays excluded. The clock pauses 20:00→09:00. |
| **Resolution SLA** | As a system, I should measure closure against a flat 48-hour target for every chat, regardless of category or customer type. |
| **Priority Ranking** | As an advisor, I should see my chats ranked by a score built from three inputs only — aging, time since last action, and SLA consumed — with the reason stated in words on every row. Refer **Annexure A2**. |
| **At-Risk Flagging** | As an advisor, I should see a chat marked At Risk on either hard condition, regardless of score. Refer **Annexure A3**. |
| **Action Pending** | As an advisor, sending a reply should never clear the chat from my plate. Only **Close** or **Not on me** does. Ticking *Requires my action* at send keeps it in Action pending. Refer **Annexure A4**. |
| **Holding Reply** | As a system, I should detect a filler-only last reply and hold the chat in Action pending with a banner quoting the message back. Fixed phrase list, not NLP. Refer **Annexure A4**. |
| **Closure** | As an advisor, I should close a chat with a required disposition. Closing over a holding reply is allowed but flagged for review. Refer **Annexure A5**. |
| **Reopen** | As an advisor, if a client writes on a chat I closed, it should reopen into **Open sessions** with the earlier disposition kept as history. Refer **Annexure A5**. |
| **Ownership** | As an advisor, I should see Primary and Secondary chats together by default, filterable by a chip control, with secondary chats tagged in the list. |

## 1.5 Annexures — Tax Advisor

### Annexure A1 – Status Model & Derivation

| Status | Derivation Rule | Means |
|---|---|---|
| **New chats** | Client sent the last message | Client is waiting on a reply |
| **Action pending** | Advisor sent last **AND** (flagged *Requires my action* **OR** holding reply detected) | Ball is on the advisor |
| **Open sessions** | Advisor sent last, nothing flagged — **or** a reopened chat | Answered, not yet closed |
| **Closed** | Disposition recorded | Finished |

**Tabs:** All · New chats · Open sessions · Action pending.

### Annexure A2 – Priority Calculation

```
score = ( aging + 2 × recent action ) × approaching SLA
```

| Input | Definition | Effect |
|---|---|---|
| Aging | Chat created → now | Base weight |
| Recent action | Last action → now | Double weight — an untouched chat is the real risk |
| Approaching SLA | Share of the applicable SLA consumed | 1× under 50% · 1.5× under 75% · 2.5× under 90% · 4× beyond |

**Bands:** At Risk ≥ 150 · Less Risky 60–149 · Later below 60.

**Actions that reset the recent-action clock:** response sent · task created · marked action pending · document requested · session closed.

#### Worked Example

| Step | Rahul | Aditya |
|---|---|---|
| Aging | 30h | 30h |
| Recent action | 26h | 2h |
| Base `aging + 2 × recent` | 30 + 52 = **82** | 30 + 4 = **34** |
| Approaching SLA (30h of 48h = 63%) | ×1.5 | ×1.5 |
| **Score** | **123** | **51** |
| **Band** | Less Risky | Later |

Same age, same SLA. The only difference is that Rahul has not been touched in 26 hours.

### Annexure A3 – At-Risk Conditions

A chat is At Risk on either condition, with no score involved:

| Condition | Test |
|---|---|
| Chat not closed | Age exceeds the 48h close target |
| Response not sent | No advisor message at all, and waiting exceeds 12 business hours |

### Annexure A4 – Action Pending & Holding Reply

| Trigger | Result |
|---|---|
| Plain reply | → Open sessions |
| `Requires my action` ticked at send | → Action pending |
| Holding reply detected | → Action pending + in-chat banner |

- **Holding reply** = the advisor's last message is filler only — "ok", "will check", "noted", "will revert".
- Aging does **not** reset on an action-flagged reply.
- Exit: **Close**, or **Not on me**.

### Annexure A5 – Closure & Reopen

**Disposition options:** Query Resolved · Task Completed · No Action Needed · Escalated · Duplicate.
Comment mandatory on Escalated and Duplicate.

| Reopen Step | Behaviour |
|---|---|
| Client writes on a closed chat | Chat reopens automatically |
| Where it lands | **Open sessions** — an existing session resuming, not a new chat |
| Prior disposition | Kept as history, never overwritten |
| Clocks | Both restart from the reopen |
| Marking | `reopened` tag on the row, plus a banner naming the earlier disposition |

A reopened chat is itself a signal — it may mean the earlier closure was premature.

### Annexure A6 – Why Closure, Not First Response

| Item | Current Model | Consequence |
|---|---|---|
| Measured on | First response time | "ok, will check" passes the SLA |
| Rewards | Replying | Advisor looks compliant while work stalls |
| Blind to | Whether anything was finished | Filing deadlines slip silently |

**Trade-off, stated plainly:** a genuinely finished chat stays visible until closed. That is the cheaper mistake — letting a reply silently clear the flag recreates the original problem.

---
---

# Part 2 — Admin

## 2.1 Problem

| Item | Description |
|---|---|
| **Target User** | Prosperr tax admin, responsible for a team of advisors. |
| **Pain Point** | Learns that a chat is at risk from the client, not from the system. |
| **User Context** | The admin dashboard today shows Document Approval, CSR Tasks and First Call. It carries **no chat information at all**. |
| **Reason** | There is no chat-level rollup per agent, and the first-response SLA reads green while chats rot unclosed. |
| **Consequence** | Cannot tell which agent is carrying risk or how much chat work is outstanding. Problems surface only after a client escalates. |

## 2.2 Hypothesis

| Item | Description |
|---|---|
| **Expected Behaviour** | If the admin dashboard shows outstanding chat load and which agents hold at-risk chats, the admin spots the drift before the client raises it. |
| **Reason** | The admin view is a rollup of the **same** model the advisor sees — same status, same at-risk rule — so the two never disagree. Visibility comes before intervention: we want to see what admins do with the numbers before building levers on an uncalibrated model. |

## 2.3 Scope

| Item | Details |
|---|---|
| **In Scope** | • **Open Chats** widget — total, split into response pending and waiting for closure<br>• **Agents at risk** widget — agents holding at-risk chats, the condition that fired, and open item count<br>• Both widgets link to the **existing agent grid** |
| **Out of Scope** | • Nudge, reassign, escalate — *Phase 2*<br>• Drill from agent into their individual chats — *Phase 2*<br>• Per-agent performance and trend lines — *Phase 3–4*<br>• Any change to the existing agent grid or the three existing dashboard widgets |

## 2.4 Acceptance Criteria

| Feature | Acceptance Criteria |
|---|---|
| **Open Chats Widget** | As an admin, I should see total open chats split into *response pending* and *waiting for closure*. The two figures must add up to the total. Refer **Annexure B1**. |
| **Agents at Risk Widget** | As an admin, I should see only agents holding at-risk chats, each showing which condition fired and their open item count. Refer **Annexure B2**. |
| **Navigation** | As an admin, clicking either widget should take me to the existing agent grid, unmodified. |
| **Consistency** | As a system, admin counts must reconcile exactly with advisor counts — same status and at-risk definitions, no separate metric path. |
| **No Intervention** | As a system, I should expose no nudge, reassign or escalate control in Phase 1. |

## 2.5 Annexures — Admin

### Annexure B1 – Dashboard Widgets

Existing widgets are unchanged. Two are added.

| Widget | Content |
|---|---|
| **Open Chats** | Total, split into *response pending* (New chats + Action pending) and *waiting for closure* (Open sessions) |
| **Agents at risk** | Agent name, condition that fired, open item count |

### Annexure B2 – Agent At-Risk Definition

```
agent_at_risk = count(chat_at_risk) > 0
```

Where `chat_at_risk` is the advisor-side definition from **Annexure A3** — chat not closed in time, or no response sent. One rule, two altitudes.

| Condition | Displayed as |
|---|---|
| Chat not closed | "2 past close target" |
| Response not sent | "3 no response sent" |

The widget shows the breakdown, not just a count, so the admin knows which condition fired.

---
---

# Common

## Assumptions

- Customer type (Basic / Advanced / Premium / Elite) is reliably present on every chat.
- Automated and system messages are identifiable and never count as the last human message.
- Chats are assigned to an advisor before they reach this screen.
- SLA targets and priority thresholds are configuration, not constants.
- The existing agent grid and customer profile screens remain unchanged.

## Phasing

| Phase | Persona Focus | Problem Solved |
|---|---|---|
| **1** | Advisor (primary), Admin (read-only) | Advisor: which chat first, replies that don't resolve, no closure. Admin: no view of chat load or agent risk. |
| **2** | Admin (primary), Advisor | Admin can see the problem but cannot act. Advisor: closure can be claimed while work is still open — task-gated closure, tasks, documents, meetings. |
| **3** | Admin | Model is hand-tuned and static; no quality check on how chats were closed. |
| **4** | Admin + Ops | Assignment and review are manual. No trend, capacity or forecasting. |

## Open Questions

| # | Question | Owner |
|---|---|---|
| 1 | A third at-risk condition was mentioned but not named. Two are defined. | Product |
| 2 | Sunday handling — the response clock excludes Saturdays and pauses 20:00–09:00; Sunday was not stated. | Product |
| 3 | Is status derived on read, or materialised with a recompute job? | Engineering |
| 4 | Is the holding-reply phrase list config or code? Preference is config. | Engineering |
| 5 | Is customer type reliably present on every chat today? | Engineering |

---

*Prototype: [`index.html`](index.html) — clickable, advisor and admin views.*
