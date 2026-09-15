# Architecture & Node Reference

Covers `02-deal-risk-system-final.json` (32 nodes) and `03-shared-error-handler.json` (4 nodes).
Each node is described as **What / Why / How**, with its inputs, outputs and downstream role.

---

# Part 1 — Architecture

## Execution flow

```
Schedule Trigger (daily 08:00)
  └─► HubSpot - Read Open Deals
        └─► Data Table - Get Deal Tracking        (close-date history, bulk read)
              └─► Data Table - Get Alert State    (alert history, bulk read)
                    └─► Code - Deal Risk Engine
                          │
                          ├─(1)─► Code - Build Run Summary ──► Sheet - Risk Scan Summary
                          ├─(2)─► Filter - Scored Deals ─────► Sheet - Deal Risk Assessment
                          ├─(3)─► Code - Build Tracking Upserts ──► Data Table - Upsert Deal Tracking
                          ├─(4)─► Code - Reap Stale State ───┬──► Sheet - Resolved Logs
                          │                                   └──► Data Table - Delete Alert Row
                          └─(5)─► Code - Route Decision ─────► Switch - Route
                                                                 │
        ┌────────────────────────────────────────────────────────┤
        │                                                        │
   [NEEDS_AI]                [REFRESH]          [RESOLVED]              [NOOP]
        │                        │                   │                     │
  AI - Revenue Risk Analyst   Sheet - Action    Sheet - Status Update    (terminal)
   ├─ Gemini 2.5 flash lite   Queue (No-AI)     Sheet - Resolved Logs
   └─ Parser - Strict JSON                      Data Table - Delete Alert Row
        │
  Code - Merge AI
    ├──► Sheet - RevOps Action Queue        (durable record, written first)
    └──► Switch - Risk Severity
           ├─ CRITICAL ─► Slack - Critical Risk Alert ─┐
           ├─ HIGH ─────► Slack - High Risk Alert ─────┤
           └─ fallback ─► Slack - Risk Routing Exception┤
                                                        ↓
                                        Code - Confirm Delivery
                                          ├──► Sheet - Alert Log
                                          └──► Data Table - Upsert Alert State

  ~20 nodes route their error output ──► Code - Aggregate Error ──► Slack - Workflow Error Alert
```

## Phases

| Phase | Nodes | Purpose |
|---|---|---|
| **1 — Trigger & retrieval** | Schedule Trigger, HubSpot | Daily cadence; defines what gets assessed |
| **2 — State loading** | Get Deal Tracking, Get Alert State | The workflow's memory: where close dates were, what was already said |
| **3 — Assessment** | Deal Risk Engine | One node derives every value the rest of the workflow uses |
| **4 — Reporting & maintenance** | Run Summary, Filter, Tracking Upserts, Reaper | Four parallel branches that don't affect alerting |
| **5 — Decision** | Route Decision, Switch - Route | The deduplication brain; four mutually exclusive outcomes |
| **6 — AI narration** | AI chain, Merge AI | One LLM call per changed deal, producing one field |
| **7 — Persistence & notification** | Action Queue, Switch - Risk Severity, Slack ×3, Confirm Delivery, Alert Log, Upsert Alert State | Record, then notify, then record state only on confirmed delivery |
| **8 — Resolution** | Status Update, Resolved Logs, Delete Alert Row | Two producers: recovered deals and departed deals |
| **9 — Error handling** | Aggregate Error, Workflow Error Alert | One message per failing node, naming affected deals |

## Three decisions that shape everything

1. **Deterministic scoring, AI narration.** Nothing downstream reads a severity from the model.
2. **Compare against the last *alerted* state.** A deal drifting 44 → 47 → 50 alerts on the third run, not never and not three times.
3. **Record state only after delivery is confirmed.** A failed Slack send means re-alert tomorrow, not permanent suppression.

## Ordering is behaviour, not decoration

In `executionOrder: v1`, when one output feeds several nodes, n8n runs them in canvas position order — top to bottom. Five such groups exist, and three carry real guarantees:

| Group | Guarantee |
|---|---|
| `Code - Merge AI` → Action Queue, then Switch Severity | Durable record written **before** Slack fires |
| `Code - Confirm Delivery` → Alert Log, then Upsert State | Delivery logged **before** dedup state is committed |
| `Switch - Route` (RESOLVED) → Status Update, Resolved Log, then Delete | State row deleted **last**, so a sheet failure leaves it intact |

Moving these nodes vertically past one another changes behaviour.

---

# Part 2 — Node reference

## Phase 1 — Trigger and retrieval

### `Schedule Trigger`

**What:** Starts one run each morning at 08:00 in the workflow timezone.
**Why:** Deal risk is a daily review rhythm, matched to a manager's morning pipeline check. Without it nothing runs — there is no other entry point.
**How:** Daily interval with `triggerAtHour: 8`. Emits a single empty item as a "go" signal; carries no data.
**Input:** None. **Output:** One empty item → `HubSpot - Read Open Deals`.
**Note:** No catch-up. If n8n is down at 08:00, that day is skipped.

### `HubSpot - Read Open Deals`

**What:** Pulls every open deal with the 15 properties the scoring model needs.
**Why:** The CRM is the source of truth for deal facts. Without this there is nothing to assess.
**How:** CRM search filtered on `hs_is_closed = false` — a computed property that works across every pipeline, unlike excluding the `closedwon`/`closedlost` stage IDs, which only match the default pipeline. `returnAll: true` pages the full result set. Retries 3× at 5s, then routes failures to its error output.
**Important:** `alwaysOutputData: true` — on a zero-deal morning it emits one empty item so the run-summary branch still executes and records `deals_scanned: 0`. The engine's deal-ID guard is what makes that safe. `num_associated_contacts` supplies the single-threading signal without an associations API call.
**Output:** One item per deal → `Data Table - Get Deal Tracking`. Errors → error sink.

### `Data Table - Get Deal Tracking`

**What:** Loads the whole `deal_tracking` table — last close date and running push count per deal.
**Why:** Push detection must watch the **whole pipeline**, because a push is what *causes* risk. Watching only flagged deals would always detect the slip too late. Remove this and the push count is permanently zero and the signal dies silently.
**How:** Unfiltered `get`, `executeOnce: true`, `returnAll: true`. `alwaysOutputData` covers the first run when the table is empty.
**Output:** One item per tracking row → `Data Table - Get Alert State`; read by name inside the engine.

### `Data Table - Get Alert State`

**What:** Loads the whole `deal_alert_state` table — what the system last told a human about each deal.
**Why:** The anti-noise mechanism. Remove it and every at-risk deal alerts every morning until the channel is muted.
**How:** Same bulk pattern. `returnAll: true` matters most here — if the table outgrows a default page, missing rows read as "never alerted" and those deals re-alert daily. That degrades gradually and never errors.
**Output:** State rows → `Code - Deal Risk Engine` (main input) and `Code - Reap Stale State` (by name).

---

## Phase 2 — Assessment

### `Code - Deal Risk Engine`

**What:** The single source of truth. Joins three data sources and derives every value the rest of the workflow uses.
**Why:** Concentrating derivation in one place is what stops two nodes computing the same thing differently. Every threshold lives in one labelled config block at the top.
**How:** Runs once for all items. Reads alert state from its input, deals from `$('HubSpot - Read Open Deals')`, tracking rows from `$('Data Table - Get Deal Tracking')`. Per deal:

1. **Guards** — a row with no ID is skipped, so a malformed API response can't become a phantom high-risk deal.
2. **Parses dates tolerantly** — accepts ISO-8601 *and* epoch milliseconds. A strict parser would silently mark every deal as missing a close date if HubSpot changed serialisation, triggering a mass false alert on a green run.
3. **Detects close-date pushes** — compares against the stored date. Forward moves increment the count; a date pulled *in* is recorded but never counted.
4. **Scores two buckets** — `dealRiskScore` (stalls, slippage, single-threading, near-close gaps) and `dataQualityScore` (missing amount, owner, next step, close date, contacts). Final score is deal risk plus half of data quality, with the data-quality half suppressed entirely for deals under 7 days old.
5. **Bands the score** into LOW / MODERATE / HIGH / CRITICAL.
6. **Attaches previous state** — previous score, level, fingerprint and `firstAlertedAt`, so no downstream node reads the state table again.
7. **Computes the trend** — RISING / FALLING / STABLE against the score at the last *delivered* alert.

**Important:** `alwaysOutputData: true` — when zero deals score, n8n injects one sentinel item so the run-summary branch still fires. `onError: continueErrorOutput` so a crash is reported rather than killing the run silently.
**Output:** One enriched item per valid deal → five branches.

---

## Phase 3 — Reporting and maintenance

### `Code - Build Run Summary`

**What:** Collapses the scan into one rollup row.
**Why:** Leadership needs totals, not rows. It's also the only place a zero-deal day is visible — without it, "nothing was at risk" and "the workflow never ran" look identical.
**How:** Filters out the sentinel by requiring `dealId`, counts by level, sums exposure in home currency, and produces two health metrics: `dataQualityOnlyCount` (HIGH/CRITICAL deals whose genuine risk score is zero — the standing check that the re-weighting still holds) and `missingCloseDateCount` (a tripwire that spikes before any date-format failure reaches Slack). Reuses the engine's `assessedAt` so summary and detail share one clock.
**Output:** Exactly one item, always → `Sheet - Risk Scan Summary`.

### `Sheet - Risk Scan Summary`

**What:** Appends the rollup row.
**Why:** Builds the time series showing whether pipeline risk is trending, and whether the model is drifting.
**How:** `append`, 12 columns, retries 3×. Terminal.

### `Filter - Scored Deals`

**What:** Drops the empty sentinel item before the audit log.
**Why:** `alwaysOutputData` on the engine injects an empty item on a zero-deal run. Without this filter, that blank item is appended to the audit log as a meaningless row.
**How:** One condition — `dealId` is not empty, with a `|| ''` guard so undefined is handled rather than throwing.
**Output:** Real scored deals → `Sheet - Deal Risk Assessment`.

### `Sheet - Deal Risk Assessment`

**What:** Appends one full assessment row per deal per day.
**Why:** The forensic record. It's how you answer "why did the system score that deal 56 three weeks ago" without re-running anything, and the only place the deal-risk / data-quality split is stored.
**How:** `append`, 27 columns including `score_breakdown` as raw JSON. Writes to its **own spreadsheet** — Google's 10-million-cell limit is per spreadsheet and this tab is ~99% of consumption, so isolating it keeps the operational tabs working when it fills. Terminal.

### `Code - Build Tracking Upserts`

**What:** Selects only deals whose close date actually moved, and shapes them for the tracking table.
**Why:** Writing every deal every run would be one database write per open deal per day for no benefit — in steady state only a small fraction of close dates move.
**How:** Filters on the `trackingChanged` flag the engine set, emits four fields.
**Output:** Often a handful of items → `Data Table - Upsert Deal Tracking`.

### `Data Table - Upsert Deal Tracking`

**What:** Persists the new close date and push count.
**Why:** Without the write-back, the next comparison has nothing to compare against and the push signal is permanently zero.
**How:** `upsert` on `deal_id`, retries 3×. Terminal.
**Note:** Deliberately not auto-reaped. A stale row for a closed deal is inert — it's only read for deals present in the current scan — and a reopened deal correctly keeps its push history.

### `Code - Reap Stale State`

**What:** Finds alert-state rows for deals that have vanished from the pipeline, and logs them as resolved.
**Why:** The normal resolution path only catches deals still open that dropped below the threshold. A deal that closes won, closes lost, or is deleted simply stops appearing — without this its state row would live forever, the table would grow without bound, and the deal would vanish with no record.
**How:** Reads the scan from its input and alert state by name. Emits rows whose deal ID is absent from the scan **and** whose last alert is older than 30 days. Two deliberate safety rules: returns immediately if the scan found zero deals, and never touches a row younger than 30 days. Without those, one transient HubSpot outage would wipe the state table and cause a full re-alert storm the next morning. Output field names deliberately match the RESOLVED route so one sheet node serves both producers.
**Output:** Orphans tagged `NOT_IN_PIPELINE`, usually zero → `Sheet - Resolved Logs` and `Data Table - Delete Alert Row`.

---

## Phase 4 — Routing

### `Code - Route Decision`

**What:** Decides what happens to each deal: alert, refresh, resolve, or nothing.
**Why:** The deduplication brain, and the reason the system doesn't become noise. Consolidating this into one node means one place to read the rule and one place to change it.
**How:** Pure routing — every value it reads was already computed by the engine. Per deal:

- **Not at risk, no prior alert** → `NOOP`.
- **Not at risk, previously alerted** → `RESOLVED`, with before/after levels and scores and `daysAtRisk`.
- **At risk** → alert only if something *material* changed: no previous state (`NEW_RISK_ALERT`), the level moved, the issue set changed, or the score moved 5+ points. Otherwise `REFRESH` — update the record, send nothing.

`firstAlertedAt` is stamped on the first alert and preserved on every subsequent one.
**Output:** Every deal, tagged → `Switch - Route`. Also read by name from `Code - Merge AI`.

### `Switch - Route`

**What:** Splits tagged deals onto four named paths.
**Why:** Makes the four possible outcomes visible on the canvas instead of buried in nested conditions.
**How:** Four rules on `$json.route` with renamed outputs, plus a fallback.
**Important:** `NOOP` is intentionally terminal — a low-risk deal never alerted needs no action. The fallback routes to the error sink as a safety net, though the field is generated from a closed set in code.

---

## Phase 5 — AI narration

### `AI - Revenue Risk Analyst`

**What:** Asks the model for one thing: the single best action to take on this deal this week.
**Why:** Every other field is deterministic. A stall count is a fact; "call the economic buyer to confirm the evaluation is still live" is a judgement, and it's the one output code can't produce. Remove this node and alerts still fire — they carry the deterministic fallback wording instead.
**How:** A system message carries the role, the authority rules (the score is computed upstream and is not the model's to restate) and the output contract. The user message wraps deal facts in an XML-style `<deal>` block with an explicit instruction that its contents are untrusted data — deal names are free text anyone can edit, so this closes the prompt-injection path. Owner ID is deliberately withheld. Batched five at a time.
**Output:** `{ deal_id, next_best_action }` per deal → `Code - Merge AI`. Failures → error sink.

### `Gemini 2.5 flash lite`

**What:** The model behind the chain. A sub-node; the chain can't run without one.
**How:** `google/gemini-2.5-flash-lite` via OpenRouter at temperature 0 for reproducible output. `maxTokens: 150` — the response is one sentence and an ID. A 60-second timeout prevents a hung call stalling the run.
**Note:** At these volumes total spend is a few dollars a year; the constraint is latency, not cost.

### `Parser - Strict JSON Schema`

**What:** Forces the reply into a strict two-field JSON object.
**Why:** Free-form output can't be joined back to a deal reliably.
**How:** Manual schema requiring `deal_id` (safe character pattern) and `next_best_action` with a minimum length approximating the word count the prompt requests. `additionalProperties: false` rejects stray fields.
**Note:** The minimum-length rule is what stops a technically-valid but useless one-word answer reaching Slack.

### `Code - Merge AI`

**What:** Joins the model's answer back onto the deals sent to it.
**Why:** **The direction of this join is the critical detail.** It iterates the *deal list*, not the AI responses. Mapping over AI output instead would mean any deal whose call failed produces no row and no alert — a critical deal could disappear on a run that looked entirely successful.
**How:** Takes the authoritative list from `$('Code - Route Decision')` filtered to `NEEDS_AI`, indexes responses by deal ID, then left-joins. A response must clear a minimum word count to be used. With no usable response it falls back to a **driver-specific** deterministic sentence — a single-threaded deal gets "identify and engage a second stakeholder this week", not generic boilerplate. Sets `actionSource` to `AI` or `FALLBACK`.
**Output:** One item per gated deal, always — never fewer → Action Queue, then Switch - Risk Severity.

---

## Phase 6 — Persistence and notification

### `Sheet - RevOps Action Queue`

**What:** Writes the full 18-column row for a deal being alerted on.
**Why:** The durable work surface. Slack messages scroll away; this row persists until someone acts.
**How:** `appendOrUpdate` on `deal_id`, so re-alerting updates the row rather than creating a second. Positioned above the alert branch so the record is written **before** Slack fires.
**Note:** Any column you add that the workflow doesn't map — assignee, notes, follow-up date — survives every run untouched.

### `Switch - Risk Severity`

**What:** Routes alerts by risk level.
**Why:** Separates CRITICAL from HIGH so the two can be styled differently and later sent to different channels without restructuring.
**How:** Two rules on `$json.riskLevel` plus a fallback. The fallback is unreachable in normal operation, which is exactly what makes it a good safety net.

### `Slack - Critical Risk Alert`

**What:** Posts the CRITICAL alert to the RevOps channel.
**Why:** The point of contact with a human. Everything upstream exists to make this message worth reading.
**How:** Eight elements in triage order: severity marker, deal name as a **clickable HubSpot link**, owner, amount, score with level and trend, close status in plain language ("12 days overdue"), why it fired now, **top three** risk drivers, and the recommended action. Drivers are capped at three deliberately — a five-bullet message reads as a report, not an alert. Every array access is guarded so a missing field can't throw and lose the alert.
**Output:** Slack API response → `Code - Confirm Delivery`. Errors → sink.
**Note:** Uses the channel ID, not the name, so renaming the channel can't silently break delivery.

### `Slack - High Risk Alert`

Same message for HIGH deals, kept as a separate node so the two can diverge — different channel, mention, or cadence — without touching the rest of the workflow.

### `Slack - Risk Routing Exception`

**What:** Catches any deal reaching the severity switch that isn't HIGH or CRITICAL.
**Why:** A safety net for a state that should be impossible. Without it such a deal would vanish without trace.
**How:** Posts to the **error** channel, not the RevOps channel — an unexpected risk level is an engineering problem, not a business review item. All fields `?? 'Unknown'`-guarded, because by definition the data is unexpected.

### `Code - Confirm Delivery`

**What:** The gate between "alert sent" and "alert recorded".
**Why:** This node prevents the worst failure the system can have. If the state upsert ran as a *sibling* of the Slack branch, a failed send would still record "alerted" — and the dedup gate would then suppress that deal on every future run. A CRITICAL alert could be lost permanently and invisibly.
**How:** Runs once per item, on Slack's success output only. The Slack node returns the API response rather than the deal, so the deal is recovered through n8n item linking via `$('Code - Merge AI').item`. If that link can't be established it **throws deliberately** — not recording state means the deal re-alerts next run, which is the safe direction; guessing would record the wrong deal. Determines the delivery channel from the originating node name with a severity-based fallback.
**Output:** One row carrying both Alert Log fields and the state-upsert payload → Alert Log, then Upsert Alert State.

### `Sheet - Alert Log`

**What:** Appends one row per **delivered** alert.
**Why:** Proof of delivery. Without it, "was this deal ever actually alerted?" requires archaeology through Slack history. It's also the raw material for tuning — counting `alert_reason` values tells you empirically whether the 5-point threshold is right.
**How:** `append`, 11 columns including `action_source`, recording whether the recommendation came from the model or the fallback. Positioned above the state upsert so the delivery record is written before the dedup state that will suppress future alerts.

### `Data Table - Upsert Alert State`

**What:** Records what was just communicated: level, score, issue fingerprint, timestamps.
**Why:** This is what tomorrow's run compares against. Remove it and every at-risk deal alerts every day forever.
**How:** `upsert` on `deal_id`. `first_alerted_at` is carried through from `Code - Route Decision` rather than recomputed, so it survives every subsequent upsert and keeps `days_at_risk` honest. Timestamps are UTC ISO, directly comparable with `assessedAt`.

### `Sheet - RevOps Action Queue (No-AI Update)`

**What:** Refreshes the queue row for a deal still at risk where nothing material changed.
**Why:** Keeps the numbers current — days to close, inactive days, score drift — without spending an AI call or sending a message.
**How:** `appendOrUpdate` with 17 of 18 columns. It **deliberately omits `recommended_action`** so the previous recommendation survives the refresh.

---

## Phase 7 — Resolution

### `Sheet - RevOps Action Queue (Status Update)`

**What:** Closes out the queue row when a deal is no longer at risk.
**Why:** Without it, resolved deals sit in the working queue forever and managers lose trust in it.
**How:** `appendOrUpdate` writing only the fields that describe the resolution. The unmapped columns — risk drivers, recommended action, the at-risk snapshot — are left untouched, because `appendOrUpdate` preserves unmapped columns. That's what keeps the record of *why* the deal was flagged.

### `Sheet - Resolved Logs`

**What:** Appends one immutable row per resolution event.
**Why:** The queue shows current state; this shows history. It's where `days_at_risk` accumulates into a measure of whether flagged deals are being resolved faster over time.
**How:** `append`, 12 columns capturing before and after state. Has **two producers** — the `RESOLVED` route (reason `RISK_IMPROVED`) and `Code - Reap Stale State` (reason `NOT_IN_PIPELINE`) — which is why both emit identical field names. The node therefore runs twice per execution, once per inbound branch.

### `Data Table - Delete Alert Row`

**What:** Removes the alert-state row so the deal starts clean if it becomes risky again.
**Why:** Without the delete, a recovered deal would still look "already alerted", and a genuine new problem months later would be suppressed as a duplicate.
**How:** `deleteRows` filtered on `deal_id`. Serves both producers. Positioned **last** on the resolution branch so the sheet writes complete first — if a sheet write fails, the state row survives and the deal is re-evaluated next run, which is the safe direction.

---

## Phase 8 — Error handling

### `Code - Aggregate Error`

**What:** The single error sink. Collapses any number of failed items into one message.
**Why:** It replaced five per-item formatters. Under the old design a rate-limited batch of 40 deals produced 40 Slack messages, which tripped Slack's own rate limit and turned one incident into two — and each message named the node but never the deals, so nobody could tell what had been skipped.
**How:** Identifies the failing node, extracts each error message, and **names the affected deal IDs**. Caps at 20 lines with an "and N more" footer.
**Input:** Error outputs from around twenty nodes. **Output:** One summary item.

### `Slack - Workflow Error Alert`

**What:** Posts the aggregated failure to the error channel.
**Why:** Makes failures visible to whoever maintains the workflow, separately from the RevOps audience.
**Important:** `onError: stopWorkflow` — **deliberately the only node that doesn't continue on error.** Every other node continues so one failure can't take down the run. But if they all did, the execution would show green no matter what happened. This node failing turns the execution red, which is the only signal that something is genuinely wrong.

---

# Part 3 — Shared Error Handler

A separate workflow, set as the Error Workflow on the main one. Four nodes.

```
Error Trigger → Code - Build Failure Report → Slack - Workflow Failure Alert
                                                   └─[on error]→ Email - Failure Fallback
```

### `Error Trigger`

**What:** Fires when a workflow naming this one as its Error Workflow fails with an **unhandled** error.
**Why:** The inline sink covers failures the workflow can catch and continue through. This covers the rest: a node throwing, an execution timing out, a trigger breaking. Without it, those failures sit red in the execution list with nobody notified.

### `Code - Build Failure Report`

**What:** Flattens the Error Trigger payload into a notification-ready shape.
**Why:** n8n emits **two different payload shapes** and most handlers only handle one. An execution failure gives `payload.execution` with an ID, URL and `lastNodeExecuted`. When the *trigger itself* fails — expired credentials, for instance — you get `payload.trigger` instead, with no execution object at all. Reading only `payload.execution` produces a blank report in exactly the situation where you most need to know.
**How:** Handles both shapes, degrades to "Unknown workflow" on a sparse payload, and clips stack traces to 1,200 characters — an unclipped trace can exceed Slack's limits and make the *failure notification* itself fail.

### `Slack - Workflow Failure Alert`

**What:** Primary notification.
**How:** Retries 3× at 5s before escalating, so a transient blip doesn't trigger email. Only on genuine failure does output 2 hand over to email.

### `Email - Workflow Failure Fallback`

**What:** Out-of-band fallback on a different transport.
**Why:** A Slack outage must not swallow the only alert.
**How:** Reads the report via `$('Code - Build Failure Report').first()` rather than from the Slack error item, so it doesn't depend on the shape of an error payload. Left on the default `stopWorkflow`: if email fails too, this run goes red, which is the last signal available.
