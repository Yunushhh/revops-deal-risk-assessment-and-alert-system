# RevOps Deal Risk Alert System

An n8n automation that scores every open HubSpot deal each morning, explains *why* a deal is at risk, maintains an action queue in Google Sheets, and sends Slack alerts only when something has genuinely changed.

This repository contains two versions of the same project: the working prototype I built first, and the optimized version it became. The point of keeping both is to show the reasoning between them.

---

## The business problem

Sales teams can't manually inspect every open opportunity. A manager with 200 deals in the pipeline cannot check each one for stalled activity, slipping close dates, or missing next steps — so deals go quiet and nobody notices until the forecast misses.

The CRM already holds the signals. What it doesn't do is turn them into a prioritised, explained list of the handful of deals that need attention *today*. HubSpot can tell you a deal hasn't been touched in 45 days; it can't tell you that this particular deal is the one worth interrupting your morning for, and why.

That gap — between raw CRM fields and an actionable decision — is what this system fills.

---

## What the system does

```
HubSpot  →  Load prior state  →  Score & classify  →  Decide what changed
                                                            ↓
                          ┌─────────────────────────────────┼──────────────────┐
                          ↓                                 ↓                  ↓
                   Audit log & summary            Action queue + Slack     Resolution
                    (Google Sheets)                (only if changed)      (close the loop)
```

Every weekday morning the workflow:

1. **Pulls all open deals** from HubSpot with the properties the risk rules need.
2. **Loads its own memory** — what it last told a human about each deal, and where each deal's close date used to be.
3. **Scores each deal** against a deterministic rubric covering stalled activity, overdue and repeatedly-pushed close dates, single-threading, and CRM data gaps.
4. **Compares against the last delivered alert** and decides: alert, refresh quietly, mark resolved, or do nothing.
5. **Writes the record first**, then sends Slack — so a notification failure never loses the row.
6. **Records state only after Slack confirms delivery**, so a failed send means the deal re-alerts tomorrow rather than being silently suppressed.

---

## Version 1 — the prototype

**`workflows/01-deal-risk-alert-system-prototype.json`** — 13 nodes, no custom code.

This was a working system, not a failed attempt. It ran daily, scored every open deal, wrote an action queue, and sent Slack alerts that people actually read.

I built it entirely from native n8n nodes because that's what I was confident I could reason about and debug. The risk calculation is a chain of five Set nodes:

```
Prepare Deal Data → Calculate Activity Age → Check Data Completeness
                  → Calculate Risk Score → Assign Risk Level
```

Each node does one visible thing. Flatten the HubSpot response. Turn dates into day counts. Flag missing fields. Sum the weights. Band the score. You can click any node and see exactly what it produced, which is genuinely useful when you're still learning.

**What it proved:** the business logic worked. The signals were the right signals, the weighting was roughly sensible, and a scored, explained queue was more useful to a manager than a CRM view.

**What it couldn't do:**

- **No memory.** It appended to the action queue every run, so the same deal reappeared every morning and the sheet filled with duplicates.
- **No deduplication.** Slack fired for every at-risk deal every day, which is how alerting systems get muted.
- **No resolution.** A deal that recovered stayed in the queue forever.
- **No error handling.** Any API failure stopped the run with no notification.
- **Scoring weighted toward data quality.** Verified by replaying the expressions: a deal with no activity for 90 days scored 25 (MODERATE, no alert), while a brand-new lead with four empty fields scored 50 (HIGH, alerted). The rubric was reliably catching CRM hygiene and staying quiet about stalled revenue.

That last point is the one I'd have missed without a structured review, and it's the single most valuable thing that came out of the next stage.

---

## Version 2 — from prototype to optimized

**`workflows/02-deal-risk-alert-system-optimized.json`** — 32 nodes, 8 Code nodes.

The prototype proved the concept but had structural limits. Five Set nodes to score one deal meant the scoring rules lived in five places; adding a signal meant touching several nodes and hoping nothing drifted. There was no way to remember what had already been said, which meant no way to be quiet.

I used AI to review the workflow end to end — node settings, expressions, execution behaviour, data flow, failure modes. The review surfaced things I hadn't considered, and a few I'd got wrong:

| Finding | Change made |
|---|---|
| Scoring rubric inverted business priority | Separated genuine deal risk from CRM data quality into two buckets, and re-weighted so a stall or a slipping date can reach HIGH on its own |
| Five Set nodes computing one score | Consolidated into one Code node where every threshold sits in a single labelled config block |
| Alerts repeated daily | Added state tracking in n8n Data Tables — alert only when the level, the issue set, or the score by 5+ points has changed |
| Alert state committed regardless of delivery | Chained the state write behind Slack's success output, so a failed send means re-alert tomorrow instead of permanent suppression |
| No resolution path | Added a resolution branch and a reaper for deals that leave the pipeline entirely |
| No failure visibility | Single error sink that aggregates failures into one message naming the affected deal IDs |
| Close dates that keep moving were invisible | Added a second Data Table tracking last-seen close date and a push counter |

JavaScript was introduced where it earned its place: joining three data sources, cross-referencing prior state, and expressing branching arithmetic over a dozen signals. Expressing that as native nodes would have needed 15+ IF/Set nodes and made the thresholds harder to audit, not easier. Where native nodes were still the clearer choice — filtering, routing, branching — they stayed.

---

## How I used AI

AI was a development assistant on this project, not a black box that produced a workflow.

**Analyse.** I gave the working prototype to an AI model and asked for a structured audit — execution settings, expressions, data flow, failure modes, business logic. It produced findings I could check rather than conclusions I had to trust.

**Understand before implementing.** For every Code block proposed, I worked through what it receives, what it returns, which nodes it replaces, and what happens when the input is empty or malformed, *before* putting it in the workflow. Where I couldn't follow the reasoning, I asked for it to be explained or simplified rather than pasting it in. Some suggestions I rejected — merging the two Slack nodes, for example, would have removed the ability to route CRITICAL and HIGH to different channels later.

**Implement.** Changes went into the workflow incrementally, with the riskiest behavioural change — the scoring re-weighting — separated from the pure reliability fixes so the two could be evaluated independently.

**Verify.** The scoring model was tested against constructed deal scenarios to confirm it behaved as intended, rather than assuming it did. That's how the inverted-priority finding was confirmed: a $5M deal stalled 45 days scored MODERATE and never alerted, which is the opposite of the system's purpose.

**What this demonstrates:** I understood the business problem and designed the workflow around it, built and ran a working prototype, used AI to find what I couldn't see myself, and made sure I understood each technical change before shipping it. I did not write the advanced JavaScript from scratch — I specified what it needed to do, reviewed what it did, and validated the result.

---

## Prototype vs optimized

| Area | Prototype | Optimized |
|---|---|---|
| **Architecture** | Linear chain: fetch → 5 transform nodes → filter → sheet + Slack | Staged: ingest → load state → score → route four ways → alert / refresh / resolve |
| **Risk logic** | One weighted score out of 100, everything in one bucket | Deal risk and data quality scored separately; hygiene contributes at half weight and is suppressed for deals under 7 days old |
| **Node count** | 13 nodes | 32 nodes — more capability, not more complexity per unit of work |
| **Coding** | 0 Code nodes; 5 Set nodes to score a deal | 8 Code nodes; **1** node to score a deal, with all thresholds in one config block |
| **Data handling** | Field-by-field through chained Set nodes | Single pass joining deals, alert state and close-date history |
| **State tracking** | None — appended a new row every run | Two n8n Data Tables: alert state and close-date tracking, both bulk-read once per run |
| **Error handling** | None — a failure stopped the run silently | Every external call routes errors to one aggregating sink that names affected deals; the final Slack node deliberately fails loudly so bad runs show red |
| **Scalability** | Action queue grew by one row per at-risk deal per day | Queue keyed on deal ID; audit log isolated in its own spreadsheet to protect the cell limit |
| **Maintainability** | Change a weight, edit several nodes | Change a weight, edit one line in one config block |
| **Alerts** | Every at-risk deal, every day | Only on material change: new risk, level change, different issues, or a 5+ point move |
| **AI usage** | None | One field — the recommended next action — with a deterministic fallback if it fails |

---

## Key design decisions

**Scoring stays deterministic; AI only writes one sentence.** Nothing downstream reads a severity from a language model. Score, level, trend and routing are all computed in code. The model produces the recommended next action and nothing else, because that's the one output deterministic logic genuinely can't produce. If the model fails, a driver-specific fallback fires instead and the alert still goes out.

**Compare against the last *alerted* state, not the last *seen* state.** A deal drifting 44 → 47 → 50 should alert on the third run — not never, and not all three times. Storing the state at the point of alert, rather than the state at last scan, is what makes that work.

**Record state only after delivery is confirmed.** In the prototype-era design the state write ran alongside the Slack send. If Slack failed, the system still recorded "alerted" and suppressed the deal from then on. Now the state write sits downstream of Slack's success output, so a failed send means the deal re-alerts tomorrow.

**Data quality is not deal risk.** A deal missing an amount is a CRM problem. A $5M deal untouched for six weeks is a revenue problem. Scoring them in the same bucket meant the second was drowned out by the first. They're now scored separately, with data quality contributing at half weight.

**Preserve the evidence.** When a deal resolves, only eight columns are rewritten. The risk drivers and the recommended action are deliberately left untouched, because Google Sheets' `appendOrUpdate` leaves unmapped columns alone — so the record of *why* the deal was flagged survives its resolution.

**Write the durable record before notifying.** In n8n's v1 execution order, parallel branches run in canvas position order. The sheet write sits above the Slack branch on purpose: if the notification fails, the row still exists.

---

## What the system outputs

| Output | Where | Contents |
|---|---|---|
| **Action queue** | Google Sheets | One row per at-risk deal: owner, amount, score, level, trend, drivers, recommended action, days at risk, status |
| **Slack alert** | Slack | Deal name linked to HubSpot, owner, amount, score with level and trend, close status, why it fired, top 3 drivers, recommended action |
| **Assessment log** | Google Sheets | One row per deal per day, including the score breakdown — makes the model auditable |
| **Alert log** | Google Sheets | One row per *delivered* alert — proof of what was actually sent |
| **Resolution log** | Google Sheets | One row per resolution, distinguishing a deal that recovered from one that left the pipeline |
| **Run summary** | Google Sheets | Daily counts by risk level, total exposure, and two health metrics that flag if the model drifts |

---

## Example

What a manager sees in Slack:

```
🚨 CRITICAL DEAL RISK

Deal: Acme Corp - Platform Expansion
Owner: 4471029
Amount: 250,000
Risk: 75/100 (CRITICAL) · RISING
Close: 12 days overdue
Why now: RISK_LEVEL_CHANGED

Risk drivers:
• Only one contact associated, so the deal depends on a single relationship.
• No sales activity for 45 days, deal is stalling.
• Close date 12 days overdue, indicating forecast slippage.

Recommended action: Identify and engage a second stakeholder this week before advancing the deal.
```

Compare that with "this deal is at risk." The alert answers which deal, whose, how much is exposed, how serious, what specifically is wrong, **what changed since last time**, and what to do about it. `RISING` tells the manager the deal is getting worse, not just that it's bad — those need different responses. And because of the deduplication gate, this message only appears when something actually changed, which is why the channel stays worth reading.

---

## Technology

- **n8n** — workflow orchestration, Code nodes, Data Tables
- **HubSpot CRM** — deal data via the CRM search API (read-only)
- **Google Sheets** — action queue, audit log, alert log, resolution log, run summary
- **Slack** — alert delivery
- **JavaScript** — in n8n Code nodes, for scoring, state comparison and data joins
- **Google Gemini via OpenRouter** — one field per alerted deal
- **AI-assisted development** — used for workflow analysis and code implementation, reviewed before use

---

## Repository structure

```
revops-deal-risk-alert-system/
├── README.md
├── workflows/
│   ├── 01-deal-risk-alert-system-prototype.json     # v1 — native nodes, no code
│   └── 02-deal-risk-alert-system-optimized.json     # v2 — production version
└── docs/
    ├── architecture-and-node-reference.md           # every node: what, why, how
    └── setup.md                                      # import and configuration
```

Both workflow files import into n8n directly. Credentials and account-specific IDs have been replaced with placeholders — see `docs/setup.md`.

---

## What this project demonstrates

- Translating a real revenue-operations problem into an automation with a defined output
- Designing a risk model around business signals rather than available data fields
- Building and running a working system in n8n using native nodes
- Recognising the limits of a first design, and knowing which limits mattered
- Using AI to analyse an existing system and implement improvements, with the judgement to evaluate what it produced
- Understanding n8n execution behaviour: item handling, execution order, error outputs, state persistence
- Reading and reasoning about JavaScript well enough to verify what it does before shipping it
- Testing business logic against constructed scenarios instead of assuming it works

---

## Limitations and future work

**Known limitations**

- Runs on a daily schedule. A deal that goes wrong at 09:00 isn't surfaced until the next morning.
- Risk signals come from HubSpot properties only. Engagement quality, email sentiment and call content would need conversation-intelligence data the system doesn't have.
- The assessment log grows by one row per deal per day. It's isolated in its own spreadsheet for headroom, but at high deal volume it will eventually need rotation.
- Owner is stored as a HubSpot ID rather than a name, which keeps the workflow to a single API call but makes the sheets less readable.
- Close-date push counts only become meaningful after a few weeks of history has accumulated.

**Possible next steps**

- Track whether the recommended action was actually taken — the most consistent recommendation in pipeline-review practice, and currently the system's biggest blind spot
- Add time-in-stage as a signal alongside activity age
- A weekly digest for resolved deals, to close the loop without adding per-event Slack noise
- Per-owner risk reporting from the assessment log
