# Advisor Chat Console – Prosperr

**Owner:** Product Team
**Status:** Draft for engineering review
**Priority:** High
**Scope of this document:** Chat workflow only. Advisor and Admin personas.

---

## Problem

| Item | Description |
|---|---|
| **Target Users** | Prosperr tax advisors and tax admins (internal CMS users). |
| **Pain Point** | 49% of client chats received zero advisor reply, ever. Of those answered, almost none are ever closed. |
| **User Context** | Advisors work a mixed book across ITR filing, advance tax, notices and planning. They triage by scrolling a raw inbox — nothing tells them which chat is closest to breaching. |
| **Reason** | SLA is measured on **first response only**, so a holding reply ("ok, will check") fully satisfies it. There is no reliable closure signal, so no one can tell an answered chat from a finished one. |
| **Consequence** | Clients wait, filing deadlines slip, and neither advisor nor admin can see which chats are at risk. Open chats grew ~47% Jun→Jul while volume grew only ~17% — neglect compounds in peak season. |

---

## Hypothesis

| Item | Description |
|---|---|
| **Expected Behaviour** | If chat status is **derived** from message and action state rather than self-declared, and chats are ranked by aging + time since last action + SLA proximity, advisors will open the right chat first and close what they finish. |
| **Reason** | Advisors are rewarded for replying and never asked to finish. Making closure the measured outcome — and making a reply insufficient to clear ownership — targets that root cause directly. Refer **Annexure D**. |

---

## Scope

| Item | Details |
|---|---|
| **In Scope** | • Derived chat status: New chats / Open sessions / Action pending / Closed<br>• Two SLA clocks — first response 12 business hours, resolution 48 hours<br>• Priority scoring from aging, recent action and SLA proximity<br>• Action-pending flag and holding-reply detection<br>• Closure with disposition capture<br>• Reopen behaviour when a client writes on a closed chat<br>• Primary / Secondary ownership facet<br>• Admin: two dashboard widgets, read-only |
| **Out of Scope** | • Task creation and task-gated closure — *Phase 2*<br>• Admin nudge, reassign, escalate — *Phase 2*<br>• Live category risk weights and threshold calibration — *Phase 3*<br>• Routing, trend lines, capacity planning — *Phase 4*<br>• Client-facing status tracker and self-serve bot<br>• Per-topic granularity inside a single chat |

---

## Acceptance Criteria

| Feature | Acceptance Criteria |
|---|---|
| **Chat Status** | As a system, I should derive every chat's status from who sent the last message plus the action-pending flag. No status dropdown exists anywhere in the UI. Refer **Annexure A**. |
| **First Response SLA** | As a system, I should measure first response against a 12-hour target counted in business hours only — 09:00 to 20:00, Saturdays excluded. The clock pauses 20:00→09:00. |
| **Resolution SLA** | As a system, I should measure closure against a flat 48-hour target for every chat, regardless of category or customer type. |
| **Priority Scoring** | As an advisor, I should see my chats ranked by a score built from three inputs only: aging, time since last action, and how much of the SLA is consumed. Every ranked row states the reason in words. Refer **Annexure B**. |
| **At-Risk Flagging** | As a system, I should mark a chat At Risk on either hard condition, regardless of score. An agent is flagged at risk if they hold even one at-risk chat. Refer **Annexure C**. |
| **Action Pending** | As an advisor, sending a reply should never clear the chat from my plate. Only **Close** or **Not on me** does. Ticking *Requires my action* at send keeps the chat in Action pending. |
| **Holding Reply** | As a system, I should detect a filler-only last reply ("ok", "will check", "noted") and hold the chat in Action pending with an in-chat banner quoting the message back. Detection is a fixed phrase list, not NLP. |
| **Closure** | As an advisor, I should close a chat with a required disposition: Query Resolved / Task Completed / No Action Needed / Escalated / Duplicate. Comment mandatory on the last two. Closing over a holding reply is allowed but flagged for review. |
| **Reopen** | As an employee, if a client writes on a closed chat it should reopen automatically into **Open sessions**, with the earlier disposition kept as history. Refer **Annexure F**. |
| **Ownership** | As an advisor, I should see Primary and Secondary chats together by default, filterable by a chip control, with secondary chats tagged in the list. |
| **Admin Dashboard** | As an admin, I should see two additional widgets on my existing dashboard — Open Chats split by state, and Agents at risk — each linking to the existing agent grid. Nothing else changes for admin in Phase 1. Refer **Annexure G**. |

---

## Assumptions

- Customer type (Basic / Advanced / Premium / Elite) is reliably present on every chat.
- Automated and system messages are identifiable and never count as the last human message.
- Advisors work a single book; chats are assigned before they reach this screen.
- SLA targets and priority thresholds are configuration, not constants.
- The existing agent grid and customer profile screens remain unchanged.

---

# Annexure

## Annexure A – Status Model & Derivation

Status is computed, never picked.

| Status | Derivation Rule | Means |
|---|---|---|
| **New chats** | Client sent the last message | Client is waiting on a reply |
| **Action pending** | Advisor sent last **AND** (flagged *Requires my action* **OR** holding reply detected) | Ball is on the advisor |
| **Open sessions** | Advisor sent last, nothing flagged — **or** a reopened chat | Answered, not yet closed |
| **Closed** | Disposition recorded | Finished |

**Tabs:** All · New chats · Open sessions · Action pending.

---

## Annexure B – Priority Calculation

```
score = ( aging + 2 × recent action ) × approaching SLA
```

| Input | Definition | Effect |
|---|---|---|
| Aging | Chat created → now | Base weight |
| Recent action | Last action → now | Double weight — an untouched chat is the real risk |
| Approaching SLA | Share of the applicable SLA consumed | 1× under 50% · 1.5× under 75% · 2.5× under 90% · 4× beyond |

**Bands:** At Risk ≥ 150 · Less Risky 60–149 · Later below 60.

### Worked Example

| Step | Rahul | Aditya |
|---|---|---|
| Aging | 30h | 30h |
| Recent action | 26h | 2h |
| Base `aging + 2 × recent` | 30 + 52 = **82** | 30 + 4 = **34** |
| Approaching SLA (30h of 48h = 63%) | ×1.5 | ×1.5 |
| **Score** | **123** | **51** |
| **Band** | Less Risky | Later |

Same age, same SLA. The only difference is that Rahul has not been touched in 26 hours.

---

## Annexure C – At-Risk Conditions

A chat is At Risk on either condition, with no score involved:

| Condition | Test | Shown as |
|---|---|---|
| Chat not closed | Age exceeds the 48h close target | "2 past close target" |
| Response not sent | No advisor message at all, and waiting exceeds 12 business hours | "3 no response sent" |

```
agent_at_risk = count(chat_at_risk) > 0
```

The same definition drives the chat's red band and the agent's highlight — one rule, two altitudes.

---

## Annexure D – Why Closure, Not First Response

| Item | Current Model | Consequence |
|---|---|---|
| Measured on | First response time | "ok, will check" passes the SLA |
| Rewards | Replying | Advisor looks compliant while work stalls |
| Blind to | Whether anything was finished | Filing deadlines slip silently |

**With closure as the metric**

| Item | Effect |
|---|---|
| Measured on | Open → Closed |
| A reply | Does not clear ownership |
| Result | Advisor is asked to finish, not just respond |

**Stated trade-off:** a genuinely finished chat stays visible until closed. That is the cheaper mistake — letting a reply silently clear the flag recreates the original problem.

---

## Annexure E – Actions That Reset the Recent-Action Clock

- Response sent
- Task created
- Marked action pending
- Document requested
- Session closed

---

## Annexure F – Reopen Behaviour

| Step | Behaviour |
|---|---|
| Client writes on a closed chat | Chat reopens automatically |
| Where it lands | **Open sessions** — not New. An existing session resuming, not a new chat. |
| Prior disposition | Kept as history, never overwritten |
| Clocks | Both restart from the reopen |
| Marking | `reopened` tag on the row, plus a banner naming the earlier disposition |

A reopened chat is itself a signal — it may mean the earlier closure was premature.

---

## Annexure G – Admin Scope, Phase 1

Existing admin dashboard widgets are unchanged. Two are added.

| Widget | Content |
|---|---|
| **Open Chats** | Total, split into *response pending* and *waiting for closure* |
| **Agents at risk** | Agents holding at-risk chats, the condition that fired, and their open item count |

Both link to the **existing agent grid**, which is not modified. Phase 1 for admin is visibility only — no nudge, no reassign, no escalate.

---

## Annexure H – Phasing

| Phase | Persona Focus | Problem Solved |
|---|---|---|
| **1** | Advisor (primary), Admin (read-only) | Advisor: which chat first, replies that don't resolve, stale chats, no closure. Admin: no view of outstanding chat work or which agent carries the risk. |
| **2** | Admin (primary), Advisor | Admin can see the problem but cannot act — no drill into an agent's chats, no intervention. Advisor: closure can be claimed while work is still open. |
| **3** | Admin | The model is hand-tuned and static — thresholds guessed, no quality check on how chats were closed. |
| **4** | Admin + Ops | Work is assigned and reviewed manually. No trend, capacity view or forecasting. |

---

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
