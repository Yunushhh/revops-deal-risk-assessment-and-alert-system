# RevOps Deal Risk Alert System

An n8n automation that evaluates open HubSpot deals for revenue and execution risk, explains *why* a deal is at risk, maintains an operational action queue in Google Sheets, and sends actionable Slack alerts for high-priority opportunities.

This repository contains two versions of the same project: the **first working implementation I built** and the **optimized version it became**.

The point of keeping both is to show the reasoning between them — not just the finished workflow.

---

## The business problem

Sales teams cannot manually inspect every open opportunity.

A manager with a large pipeline cannot check every deal for stalled activity, slipping close dates, missing next steps, forecast risk, or execution gaps. The CRM contains many of the signals, but those signals do not automatically become a clear decision about **which deal needs attention now and why**.

This system is designed to close that gap:

> **Raw CRM data → risk assessment → explanation → action → alert**

Instead of giving a manager another list of CRM fields, the system tries to answer:

**Which deals are risky?**
**How serious is the risk?**
**Why is the deal risky?**
**What should a person review next?**

---

## What the system does

```text
HubSpot
   ↓
Load open deals
   ↓
Risk assessment
   ↓
Risk score + risk drivers
   ↓
High / Critical?
   ├── No → continue normal processing
   │
   └── Yes
         ↓
    AI business assessment
         ↓
    Action queue
         ↓
    Slack alert
```

The optimized version extends this basic idea with persistent state, alert lifecycle handling, change detection, and stronger error handling.

At a high level, the system:

1. **Reads open deals** from HubSpot with the properties required for assessment.
2. **Evaluates deterministic risk signals** such as close-date problems, inactivity, missing next steps, missing ownership, and other execution gaps.
3. **Calculates a risk score and severity** using explicit business rules.
4. **Explains the detected risk** through human-readable risk drivers.
5. **Uses AI only where it adds value** — interpreting business impact and suggesting a next action for higher-risk deals.
6. **Stores the assessment** in Google Sheets.
7. **Maintains an operational action queue** for deals requiring human review.
8. **Sends Slack alerts** for high and critical opportunities.
9. In the optimized version, **tracks state and changes over time** so alerts do not simply repeat every run.

---

## Version 1 — the first working prototype

**`workflows/01-deal-risk-monitoring-assessment-system-first-draft.json`**

This was my first substantial working implementation.

It was not a throwaway experiment and it was not designed to be artificially simple. I started with the business problem and built the workflow around the logic I believed a RevOps team would need.

The first version could:

* read open HubSpot deals
* identify multiple risk signals
* calculate a deterministic risk score
* classify risk as LOW, MODERATE, HIGH, or CRITICAL
* generate human-readable risk drivers
* calculate run-level risk statistics
* send higher-risk deals through an AI assessment
* maintain structured AI output
* write deal assessments to Google Sheets
* maintain an Action Queue
* send severity-based Slack alerts
* route some workflow failures to an operational Slack alert

The core risk engine was implemented in JavaScript because the scoring logic had already become more involved than a simple sequence of IF conditions.

The engine evaluated signals including:

* missing or invalid deal amount
* unassigned deals
* missing next steps
* missing close dates
* overdue close dates
* stalled or aging activity
* near-close execution gaps
* high-value opportunities

It then produced a score, risk level, issue codes, severity information, risk drivers, and a score breakdown.

For HIGH and CRITICAL deals, the workflow used an AI analysis step for a different purpose: the deterministic logic decided **whether the deal was risky**, while the AI was asked to interpret the business impact and recommend one concrete human action. The deterministic risk level remained authoritative.

I also used structured output validation and reattached the AI response to the original deal using the deal ID rather than relying on item position.

### What this version proved

The first version proved that I could translate a RevOps requirement into a working automation rather than simply building an API-to-Slack workflow.

It demonstrated that the system could:

**identify risk → explain the risk → prioritize it → produce an operational output.**

It also helped expose where the architecture needed to mature.

### What this version revealed

* **The implementation became code-heavy.**
  The scoring engine, run summary, AI result reconstruction, and error formatting used custom JavaScript. The logic worked, but the workflow was becoming harder to reason about as the number of responsibilities grew.

* **Risk calculation and workflow orchestration were starting to become tightly coupled.**
  The business rules were working, but the structure made later changes more expensive.

* **The workflow had limited lifecycle awareness.**
  The Action Queue could maintain deal records using the deal ID as a matching key, but the mature alert lifecycle of the optimized workflow was not yet present.

* **The AI step needed strict boundaries.**
  I did not want an LLM deciding whether a deal was HIGH or CRITICAL. The deterministic model remained the source of truth, while AI was limited to business interpretation and next-action guidance.

* **Operational edge cases needed more attention.**
  Error paths existed, but the workflow had not yet evolved into the more deliberate state, delivery, and failure-handling architecture of the final version.

The first draft therefore became more than a prototype: it became the baseline I could critically evaluate.

---

## Version 2 — from working prototype to optimized system

**`workflows/02-deal-risk-alert-system-optimized.json`**

The first version proved the business logic. The next question was:

> **Can the same business outcome be achieved with a cleaner, more reliable and more maintainable architecture?**

I used AI to review the workflow end to end — including node settings, expressions, data flow, execution behaviour, failure modes, state handling, and business logic.

The review surfaced both technical and business issues that were difficult to see from the workflow canvas alone.

| Finding                                                          | Change made                                                                                    |
| ---------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| Risk scoring mixed business risk with CRM data quality           | Separated genuine deal risk from data-quality signals and rebalanced their influence           |
| Multiple workflow steps were performing related transformations  | Consolidated suitable logic into focused Code nodes where that made the architecture clearer   |
| Alerts could repeat without meaningful change                    | Added persistent state and change detection                                                    |
| Alert state could become inconsistent with notification delivery | Changed state handling so delivery success is considered before committing the alert state     |
| No complete resolution lifecycle                                 | Added resolution handling for deals whose risk clears or whose deal leaves the active pipeline |
| Close-date movement was difficult to track                       | Added persistent close-date history and push tracking                                          |
| Failure visibility was limited                                   | Added centralized operational error handling                                                   |
| AI output needed stronger guarantees                             | Added structured output validation and deterministic fallback behaviour                        |

JavaScript was introduced where it provided a genuine architectural advantage — particularly for scoring arithmetic, state comparison, data joining, and multi-signal logic.

The goal was **not** to replace native n8n nodes just to reduce the node count.

Native nodes remained where they were clearer — for example, filtering, branching, routing and integration points.

The important change was moving from:

> **"many nodes because each small operation gets its own node"**

toward:

> **"use the right abstraction for each kind of logic."**

---

## How I used AI

AI was a development assistant on this project, not a black box that produced a workflow I blindly imported.

### Analyse

I used AI to review the working prototype at a deeper level:

* node settings
* expressions
* data flow
* execution behaviour
* business logic
* error handling
* state management
* performance
* maintainability

### Understand before implementing

When AI proposed JavaScript or architectural changes, I first worked through:

* what the code receives
* what it returns
* what business rule it implements
* which nodes it replaces
* how multiple items are handled
* what happens with missing or unexpected input

The goal was to understand the change before using it.

### Implement

Useful changes were then incorporated into the workflow incrementally.

I did not treat AI-generated code as automatically correct.

### Verify

I checked whether the resulting behaviour still matched the business requirement and whether the changes introduced new failure modes.

This distinction matters:

**AI helped with technical implementation and review. It did not replace the need to understand the workflow.**

---

## Prototype vs optimized

| Area                | First Draft                                                                   | Optimized                                                                            |
| ------------------- | ----------------------------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| **Architecture**    | Business logic implemented across several dedicated processing and Code nodes | More deliberate separation of ingestion, assessment, state, routing and notification |
| **Risk logic**      | Deterministic multi-signal score with explicit risk drivers                   | Refined scoring and clearer separation of business risk vs data quality              |
| **Coding**          | Several Code nodes handling scoring, aggregation and data reconstruction      | Code concentrated where it provides a real structural advantage                      |
| **Data handling**   | Deal data transformed and enriched through the workflow                       | More deliberate joining and state-aware processing                                   |
| **State tracking**  | Action Queue provided operational storage but had limited lifecycle awareness | Persistent state and change detection support alert lifecycle                        |
| **Alerts**          | High / Critical routing with Slack notifications                              | Alerts designed around meaningful state/change rather than simple repeated detection |
| **Error handling**  | Dedicated error-formatting and Slack error paths                              | More deliberate failure handling and visibility                                      |
| **AI usage**        | AI used for business interpretation of high-risk opportunities                | AI kept bounded to areas where deterministic logic is insufficient                   |
| **Maintainability** | Working but increasingly code-heavy as requirements grew                      | Logic consolidated and responsibilities made easier to reason about                  |
| **Scalability**     | Suitable as an initial working implementation                                 | Designed with repeated executions, state and growing history in mind                 |

---

## Key design decisions

### Scoring stays deterministic; AI does not decide severity

The system does not ask an LLM whether a deal is HIGH or CRITICAL.

Risk score, risk level, and routing remain deterministic.

AI is used for business interpretation and action guidance, where there is more value in language-based reasoning.

This creates a clear boundary:

**Deterministic logic decides. AI explains and assists.**

---

### Business risk is different from CRM data quality

A missing close date is a data-quality problem.

A high-value deal that has stalled and is approaching its expected close date is a revenue problem.

These are related, but they are not the same thing.

The optimized workflow therefore treats them deliberately instead of allowing CRM hygiene issues to dominate the overall risk signal.

---

### The Action Queue is an operational output, not a second CRM

The workflow does not attempt to reproduce every field already available in HubSpot.

The Action Queue exists to help a human answer:

> **Which deals need attention, why, and what should happen next?**

This keeps the automation focused on decision support rather than recreating the CRM.

---

### AI output is validated before it is used

The AI response is structured and validated before downstream processing.

The deterministic deal data is then reattached using the deal ID rather than trusting item position.

That means the LLM contributes its intended output without becoming the source of truth for the underlying deal record.

---

### Reliability matters more than simply reducing node count

A workflow with fewer nodes is not automatically better.

The objective of the optimization was to reduce unnecessary complexity while keeping the architecture understandable, testable and reliable.

---

## What the system outputs

| Output                   | Where                | Purpose                                                     |
| ------------------------ | -------------------- | ----------------------------------------------------------- |
| **Deal risk assessment** | Google Sheets        | Record of the assessment and supporting risk information    |
| **Action queue**         | Google Sheets        | Operational list of opportunities requiring human attention |
| **Slack alert**          | Slack                | Immediate notification for high-priority risk               |
| **Run summary**          | Google Sheets        | High-level information about the assessment run             |
| **Risk drivers**         | Sheets / Slack       | Human-readable explanation of why the deal was flagged      |
| **Recommended action**   | Action Queue / Slack | Converts detection into something a person can act on       |

The exact outputs and lifecycle behaviour become more sophisticated in the optimized version.

---

## Example

A useful risk alert should not simply say:

```text
Deal is at risk.
```

A more useful operational output is:

```text
Risk: 68 / CRITICAL

Drivers:
• 45 days inactive
• Close date in 5 days
• Missing next step

Recommended action:
Review the opportunity and establish the next concrete customer action.
```

The important idea is that a sales user should be able to understand:

**which deal → how serious → why → what to investigate next**

without opening several CRM screens first.

---

## Technology

* **n8n** — workflow orchestration
* **HubSpot CRM** — source of deal data
* **Google Sheets** — operational records, assessment data and summaries
* **Slack** — alert delivery
* **JavaScript** — selected n8n Code nodes for deterministic logic and data processing
* **LLM / AI** — business interpretation and next-action assistance
* **AI-assisted development** — workflow review, technical analysis and implementation assistance

---

## Repository structure

```text
revops-deal-risk-alert-system/
├── README.md
├── workflows/
│   ├── 01-deal-risk-monitoring-assessment-system-first-draft.json
│   └── 02-deal-risk-alert-system-optimized.json
└── docs/
    ├── architecture-and-node-reference.md
    └── setup.md
```

Credentials, account-specific IDs and other deployment-specific values are replaced with placeholders in the public workflow files.

---

## What this project demonstrates

* Translating a real RevOps problem into a working automation
* Identifying meaningful sales-deal risk signals
* Designing deterministic business rules around those signals
* Turning risk calculations into actionable outputs
* Building and testing a multi-integration workflow in n8n
* Understanding when native nodes are useful and when Code nodes provide a better abstraction
* Using AI to analyse an existing automation rather than blindly generating one
* Reading and reasoning about AI-generated JavaScript before implementing it
* Thinking about state, alert fatigue, failure handling and maintainability
* Iterating from a functioning first implementation toward a more robust design

---

## Limitations and future work

### Known limitations

* Risk signals are limited by the information available in the connected CRM.
* The workflow does not automatically know the full quality of customer conversations, sentiment, or buying-committee engagement unless those signals are available from integrated systems.
* Historical assessment data can continue to grow and may eventually require a more scalable storage approach.
* Recommended actions are suggestions for human review, not automatic sales actions.
* The system can identify risk, but it does not guarantee that a deal will be won or lost.

### Possible next steps

* Track whether the recommended action was actually taken
* Add time-in-stage as an additional signal
* Add richer engagement and conversation-intelligence signals
* Introduce owner-level and team-level risk reporting
* Add periodic summaries without increasing real-time notification noise
* Continue refining the scoring model using observed outcomes rather than assumptions

---

## Why both versions are included

The final workflow is the more technically mature version, but the First Draft is important because it shows **how the system evolved**.

The project progression is:

```text
Business problem
      ↓
First working implementation
      ↓
Structured review
      ↓
AI-assisted technical analysis
      ↓
Understanding and validating proposed changes
      ↓
Architecture optimization
      ↓
Final workflow
```

The goal of this repository is therefore not to present a workflow that appeared fully formed.

It is to show the ability to:

**understand the business problem → build a working solution → identify its weaknesses → use the right tools to improve it → understand what changed → validate the result.**
