# Setup

All three workflows import into n8n as-is. Credentials and account-specific identifiers are replaced with placeholders, so you need to attach your own before activating.

## 1. Import

n8n → **Workflows** → **Import from File** → select the JSON from `workflows/`.

Start with `01-deal-risk-system-draft.json` if you want to see the logic in its simplest form. Use `02-deal-risk-system-final.json` for the production version.

## 2. Attach credentials

No working credentials are embedded — every reference is a `GITHUB_PUBLIC_CREDENTIAL_PLACEHOLDER`. Open each node that needs one and select your own.

| Credential | Used by | Scope needed |
|---|---|---|
| HubSpot App Token | `HubSpot - Read Open Deals` | `crm.objects.deals.read`, `crm.schemas.deals.read` — read-only |
| Google Service Account | all Google Sheets nodes | Sheets access to both spreadsheets |
| Slack API | all Slack nodes | `chat:write` |
| OpenRouter API | `Gemini 2.5 flash lite` (final only) | — |
| Groq API | `Groq Chat Model` (draft only) | — |
| SMTP | `Email - Workflow Failure Fallback` (error handler only) | — |

## 3. Replace the placeholders

Search each imported workflow for `YOUR_` and replace.

| Placeholder | What to use | Where |
|---|---|---|
| `YOUR_GOOGLE_SHEETS_SPREADSHEET_ID` | Spreadsheet holding all tabs | Draft |
| `YOUR_OPERATIONS_SPREADSHEET_ID` | Action queue, alert log, resolved log, run summary | Final |
| `YOUR_AUDIT_SPREADSHEET_ID` | Separate spreadsheet for the per-deal assessment log | Final |
| `YOUR_DEAL_ALERT_STATE_TABLE_ID` | n8n Data Table storing alert state | Final |
| `YOUR_DEAL_TRACKING_TABLE_ID` | n8n Data Table storing close-date history | Final |
| `YOUR_REVOPS_RISK_CHANNEL_ID` | Slack channel for deal risk alerts | Final |
| `YOUR_WORKFLOW_ERROR_CHANNEL_ID` | Slack channel for system errors | Final, error handler |
| `YOUR_HUBSPOT_PORTAL_ID` | Your HubSpot portal ID — enables clickable deal links | `Code - Deal Risk Engine` config block |
| `YOUR_ALERT_FROM_ADDRESS` / `YOUR_ONCALL_EMAIL_ADDRESS` | Email fallback addresses | Error handler |

Use Slack **channel IDs**, not names. A renamed channel silently breaks delivery when referenced by name.

## 4. Create the Google Sheets tabs

**Draft** needs three tabs: `Deal Risk Assessment` (18 columns), `Risk Scan Summary` (9), `RevOps Action Queue` (17).

**Final** needs five. Each Google Sheets node's column mapping is the authoritative list — open the node to read the exact headers.

| Tab | Columns | Spreadsheet |
|---|---|---|
| `RevOps Action Queue` | 18, matched on `deal_id` | Operations |
| `Risk Scan Summary` | 12 | Operations |
| `Alert Log` | 11 | Operations |
| `Resolved Log` | 12 | Operations |
| `Deal Risk Assessment` | 27 | Audit (separate spreadsheet) |

Header text must match exactly. n8n maps Sheets columns by header name, not position.

The audit log lives in its own spreadsheet on purpose: Google's 10-million-cell limit is per spreadsheet, and that tab is roughly 99% of consumption.

## 5. Create the n8n Data Tables

Final version only.

**`deal_alert_state`** — `deal_id` (string), `deal_name` (string), `risk_level` (string), `risk_score` (number), `issue_fingerprint` (string), `first_alerted_at` (string), `last_alerted_at` (string)

**`deal_tracking`** — `deal_id` (string), `close_date` (string), `close_date_push_count` (number), `last_seen_at` (string)

Timestamps are stored as strings, not dates — the workflow writes UTC ISO strings and this avoids type-coercion surprises on upsert.

## 6. Wire up the error handler

1. Import `03-shared-error-handler.json`
2. Attach Slack and SMTP credentials, replace the three placeholders
3. **Activate it** — an Error Workflow won't fire while inactive
4. In the main workflow: **Settings → Error Workflow → Shared Error Handler**

Point every future workflow at the same one.

## 7. Tune before activating

Thresholds live in one config block at the top of `Code - Deal Risk Engine`:

```javascript
const THRESHOLDS          = { CRITICAL: 60, HIGH: 40, MODERATE: 20 };
const STALL_DAYS          = { LOSING: 14, STALLED: 30, SEVERE: 60 };
const HIGH_VALUE_MIN      = 100000;
const DQ_WEIGHT           = 0.5;
const NEW_DEAL_GRACE_DAYS = 7;
const TREND_DELTA         = 3;
```

## 8. First run

Run once manually with the Slack nodes disabled, then check the action queue before going live.

On the first run every at-risk deal has no prior state, so all of them qualify as new alerts — expect a burst. The first run also seeds `deal_tracking`, so close-date push counts start at zero and only become meaningful after a few weeks of history.

Set a filter view on the action queue: `action_status = REVIEW_REQUIRED`, sorted by `risk_score` descending. That view is the working queue.

## 9. Watch these two numbers

In `Risk Scan Summary`:

- **`data_quality_only_count`** — HIGH/CRITICAL deals flagged purely on CRM hygiene. If it climbs, the scoring model has drifted back toward being a data-hygiene alarm.
- **`missing_close_date_count`** — a tripwire. If a HubSpot date format ever changes, this spikes to near 100% of deals before the resulting alert storm reaches Slack.
