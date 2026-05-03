Sync Meta Ads insights to Google Sheets with backfill and weekly ETL

https://n8nworkflows.xyz/workflows/sync-meta-ads-insights-to-google-sheets-with-backfill-and-weekly-etl-14721


# Sync Meta Ads insights to Google Sheets with backfill and weekly ETL

# 1. Workflow Overview

This workflow extracts Meta Ads Insights data for two separate ad accounts and appends the results into Google Sheets. It supports both:

- a **manual historical backfill** over a custom date range, split into weekly chunks
- an **automatic incremental sync** every 7 days for the latest 7-day window

The workflow is designed as a lightweight ETL pipeline:
- extract from the Meta Graph API
- transform the API payload into row-oriented records
- load into Google Sheets tabs for reporting
- log skipped periods when the API returns no usable data or errors

It handles two account-specific branches in parallel:

- **Account A**: campaign-level weekly insights
- **Account B**: ad-level insights with daily breakdown (`time_increment=1`)

A key limitation is explicitly documented in the workflow notes: **all writes are append-only**, so rerunning the same date range will create duplicate rows unless a deduplication strategy is added externally or in the workflow.

## 1.1 Entry and Date Range Preparation

The workflow has two entry points:

- **Manual Trigger (Historical Backfill)** for one-off backfills
- **Schedule Trigger (Incremental)** for recurring weekly syncs

These entry points produce date windows that feed both account-processing branches.

## 1.2 Historical Backfill Range Chunking

For manual runs, a static start and end date are defined and then split into weekly periods. Each period becomes one item, which is processed sequentially by both account branches.

## 1.3 Incremental Weekly Range

For scheduled runs, the workflow computes a dynamic date range from “now minus 7 days” to “today” and sends that single period directly to both account branches.

## 1.4 Account A Extraction and Load

This branch loops through each period, calls the Meta Insights API at **campaign** level, validates the response, splits the returned array into individual records, filters out rows with zero spend, and appends the remaining data to the **Account_A** Google Sheet tab. If no usable data is returned, it logs the skipped period to **Account_A_Log**.

## 1.5 Account B Extraction and Load

This branch mirrors Account A, but calls the Meta Insights API at **ad** level with `time_increment=1`, producing daily ad-level rows. Valid rows with non-zero spend are appended to **Account_B**, and skipped periods are appended to **Account_B_Log**.

## 1.6 Logging and Operational Notes

Skipped periods are written to dedicated log tabs with:
- status
- reason
- account label
- date window
- execution ID
- timestamp

This provides basic traceability for periods that produced no records or encountered API issues.

---

# 2. Block-by-Block Analysis

## 2.1 Documentation and In-Canvas Notes

### Overview
These sticky notes document the workflow’s purpose, setup steps, known limitation, and the specific role of each account branch. They do not affect execution but are important for maintainability and onboarding.

### Nodes Involved
- Workflow Overview
- Sticky Note3
- Sticky Note4

### Node Details

#### Workflow Overview
- **Type and role:** Sticky Note; in-canvas documentation node.
- **Configuration choices:** Contains a full description of the workflow, setup instructions, limitations, and usage modes.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Sticky Note node version 1.
- **Edge cases or potential failure types:** None at runtime.
- **Sub-workflow reference:** None.

#### Sticky Note3
- **Type and role:** Sticky Note; documents Account A branch.
- **Configuration choices:** Notes that Account A uses `level=campaign`, weekly granularity, and writes to `Account_A`.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Sticky Note node version 1.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note4
- **Type and role:** Sticky Note; documents Account B branch.
- **Configuration choices:** Notes that Account B uses `level=ad` and `time_increment=1`, writing to `Account_B`.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Sticky Note node version 1.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

---

## 2.2 Historical Backfill Entry and Weekly Period Generation

### Overview
This block is the manual backfill path. It starts from a manual trigger, defines a fixed historical date range, and splits that range into 7-day chunks so the Meta API can be queried in smaller windows.

### Nodes Involved
- Manual Trigger (Historical Backfill)
- Set Date Range (Historical Backfill)
- Generate Weekly Periods

### Node Details

#### Manual Trigger (Historical Backfill)
- **Type and role:** `manualTrigger`; starts the workflow interactively from the editor.
- **Configuration choices:** No parameters; intended for manual use only.
- **Key expressions or variables used:** None.
- **Input and output connections:** Outputs to **Set Date Range (Historical Backfill)**.
- **Version-specific requirements:** Type version 1.
- **Edge cases or potential failure types:** None beyond normal manual execution behavior.
- **Sub-workflow reference:** None.

#### Set Date Range (Historical Backfill)
- **Type and role:** `set`; creates a JSON payload with the backfill boundaries.
- **Configuration choices:** Uses raw JSON output:
  - `start_date`: `2024-01-01`
  - `end_date`: `2024-12-31`
- **Key expressions or variables used:** Static values in the provided workflow.
- **Input and output connections:** Receives from **Manual Trigger (Historical Backfill)**, outputs to **Generate Weekly Periods**.
- **Version-specific requirements:** Set node version 3.4.
- **Edge cases or potential failure types:**
  - Invalid date strings will cause downstream issues in the Code node.
  - If `start_date > end_date`, no periods will be generated.
- **Sub-workflow reference:** None.

#### Generate Weekly Periods
- **Type and role:** `code`; splits the historical date range into weekly intervals.
- **Configuration choices:** JavaScript code:
  - reads `start_date` and `end_date` from the first input item
  - creates 7-day windows
  - clamps the final `until` date so it never exceeds `end_date`
  - emits items shaped as `{ since, until }`
- **Key expressions or variables used:**
  - `$input.first().json.start_date`
  - `$input.first().json.end_date`
- **Input and output connections:** Receives from **Set Date Range (Historical Backfill)**; outputs in parallel to:
  - **Loop Over Periods (Account A)**
  - **Loop Over Periods (Account B)**
- **Version-specific requirements:** Code node version 2.
- **Edge cases or potential failure types:**
  - Invalid date parsing can produce `Invalid Date` values and malformed output.
  - Time zone conversion via `toISOString()` may shift dates if local timezone handling is unexpected.
  - If input is missing, `$input.first()` can fail.
- **Sub-workflow reference:** None.

---

## 2.3 Incremental Scheduled Range Preparation

### Overview
This block is the automated weekly sync path. Every 7 days, it computes a single rolling date range covering the last 7 days and sends that window to both account branches.

### Nodes Involved
- Schedule Trigger (Incremental)
- Set Incremental Range (Last 7 Days)

### Node Details

#### Schedule Trigger (Incremental)
- **Type and role:** `scheduleTrigger`; launches the workflow on a recurring schedule.
- **Configuration choices:** Interval rule with `daysInterval: 7`.
- **Key expressions or variables used:** None.
- **Input and output connections:** Outputs to **Set Incremental Range (Last 7 Days)**.
- **Version-specific requirements:** Type version 1.3.
- **Edge cases or potential failure types:**
  - Execution timing depends on server timezone and activation state.
  - If the workflow is inactive, the schedule will not run.
- **Sub-workflow reference:** None.

#### Set Incremental Range (Last 7 Days)
- **Type and role:** `set`; creates a dynamic JSON payload with `since` and `until`.
- **Configuration choices:** Raw JSON output using expressions:
  - `since`: `{{ $now.minus(7, 'days').toISODate() }}`
  - `until`: `{{ $now.toISODate() }}`
- **Key expressions or variables used:**
  - `$now.minus(7, 'days').toISODate()`
  - `$now.toISODate()`
- **Input and output connections:** Receives from **Schedule Trigger (Incremental)**; outputs to:
  - **Loop Over Periods (Account A)**
  - **Loop Over Periods (Account B)**
- **Version-specific requirements:** Set node version 3.4; depends on expression support for `$now`.
- **Edge cases or potential failure types:**
  - Date interpretation depends on n8n instance timezone.
  - The range may overlap previous runs depending on exact schedule time and reporting latency.
- **Sub-workflow reference:** None.

---

## 2.4 Account A — Campaign-Level Extraction, Validation, Filtering, Load, and Logging

### Overview
This branch processes each period for Account A. It fetches campaign-level insights from Meta, checks whether the API returned a non-empty `data` array, splits the payload into row items, filters out zero-spend rows, appends valid rows to Google Sheets, and logs skipped periods when no usable data is returned.

### Nodes Involved
- Loop Over Periods (Account A)
- Fetch Meta Insights (Account A)
- Valid Response? (Account A)
- Split API Response (Account A)
- If Spend not 0 (Account A)
- Append to Sheet (Account A)
- Log Skipped Period (Account A)
- Append to Log Sheet (Account A)

### Node Details

#### Loop Over Periods (Account A)
- **Type and role:** `splitInBatches`; iterates over incoming date-period items one at a time.
- **Configuration choices:** Default options; no explicit batch size shown, so it uses standard node behavior for item iteration.
- **Key expressions or variables used:** Downstream nodes reference this node’s current item.
- **Input and output connections:**
  - Receives from **Generate Weekly Periods** and **Set Incremental Range (Last 7 Days)**
  - Output index 1 goes to **Fetch Meta Insights (Account A)**
  - Receives loop-back from **Append to Sheet (Account A)** and **Append to Log Sheet (Account A)** to continue iteration
- **Version-specific requirements:** Split In Batches version 3.
- **Edge cases or potential failure types:**
  - Incorrect loop-back wiring can stop iteration.
  - If no input items are received, nothing runs.
- **Sub-workflow reference:** None.

#### Fetch Meta Insights (Account A)
- **Type and role:** `httpRequest`; calls the Meta Graph API Insights endpoint.
- **Configuration choices:**
  - URL: `https://graph.facebook.com/v24.0/act_XXXXXXXXXXXXXXXXX/insights`
  - Authentication: Generic credential type using **HTTP Header Auth**
  - Query parameters:
    - `fields`: extensive insights field list
    - `level=campaign`
    - `time_range[since]={{ $json.since }}`
    - `time_range[until]={{ $json.until }}`
  - Pagination enabled using `paging.next`
  - `neverError: true`
  - `retryOnFail: true`
  - `maxTries: 5`
  - `waitBetweenTries: 3000`
  - `continueOnFail: true`
- **Key expressions or variables used:**
  - `$json.since`
  - `$json.until`
  - `$response.body.paging.next`
  - `!$response.body.paging?.next`
- **Input and output connections:** Receives from **Loop Over Periods (Account A)**; outputs to **Valid Response? (Account A)**.
- **Version-specific requirements:** HTTP Request node version 4.3.
- **Edge cases or potential failure types:**
  - Invalid or expired Meta token
  - Missing `ads_read` permission
  - Incorrect ad account ID in URL
  - API throttling/rate limiting
  - Pagination shape changing or `paging.next` absent
  - Because `neverError` and `continueOnFail` are enabled, API failures may pass downstream as non-throwing responses and be treated as “skipped”
- **Sub-workflow reference:** None.

#### Valid Response? (Account A)
- **Type and role:** `if`; checks whether the API response contains a non-empty `data` array.
- **Configuration choices:**
  - Boolean condition:
    `{{ Array.isArray($json.data) && $json.data.length > 0 }} == true`
- **Key expressions or variables used:**
  - `Array.isArray($json.data)`
  - `$json.data.length`
- **Input and output connections:**
  - Receives from **Fetch Meta Insights (Account A)**
  - True output goes to **Split API Response (Account A)**
  - False output goes to **Log Skipped Period (Account A)**
- **Version-specific requirements:** If node version 2.
- **Edge cases or potential failure types:**
  - If Meta returns an error object instead of `data`, it routes to the false branch.
  - Empty but valid reporting periods are logged as skipped, not distinguished from actual API failures.
- **Sub-workflow reference:** None.

#### Split API Response (Account A)
- **Type and role:** `code`; converts the `data` array into one item per returned insight row.
- **Configuration choices:**
  - Flattens all items with `flatMap`
  - Maps each row into normalized JSON
  - Converts numeric metrics with `Number(...)`
  - Defaults arrays to `[]`
  - Defaults missing ranking fields to `''`
- **Key expressions or variables used:**
  - `item.json.data || []`
  - `Number(row.spend || 0)` and similar conversions
- **Input and output connections:** Receives from **Valid Response? (Account A)** true branch; outputs to **If Spend not 0 (Account A)**.
- **Version-specific requirements:** Code node version 2.
- **Edge cases or potential failure types:**
  - If API payload format changes, fields may become undefined.
  - Arrays like `actions` remain arrays; downstream Google Sheets handling may serialize them inconsistently.
  - Numeric conversion can turn invalid strings into `NaN`.
- **Sub-workflow reference:** None.

#### If Spend not 0 (Account A)
- **Type and role:** `if`; filters out rows where `spend` equals zero.
- **Configuration choices:**
  - Number condition: `{{ $json.spend }} != 0`
  - Loose type validation enabled
- **Key expressions or variables used:**
  - `$json.spend`
- **Input and output connections:**
  - Receives from **Split API Response (Account A)**
  - True output goes to **Append to Sheet (Account A)**
  - False output is unused
- **Version-specific requirements:** If node version 2.2.
- **Edge cases or potential failure types:**
  - `NaN` handling may behave unexpectedly under loose typing.
  - Valid zero-spend rows are intentionally dropped, which may or may not match reporting requirements.
- **Sub-workflow reference:** None.

#### Append to Sheet (Account A)
- **Type and role:** `googleSheets`; appends transformed rows to the reporting sheet.
- **Configuration choices:**
  - Operation: `append`
  - Spreadsheet: `YOUR_SPREADSHEET_ID`
  - Sheet: `Account_A` (shown via gid selection)
  - Mapping mode: define below
  - Explicit field mapping for all output columns
  - Type conversion disabled
- **Key expressions or variables used:** Column expressions such as `{{ $json.date_start }}`, `{{ $json.spend }}`, etc.
- **Input and output connections:**
  - Receives from **If Spend not 0 (Account A)** true branch
  - Outputs back to **Loop Over Periods (Account A)** to continue the loop
- **Version-specific requirements:** Google Sheets node version 4.7.
- **Edge cases or potential failure types:**
  - Missing or invalid Google Sheets OAuth2 credential
  - Spreadsheet/tab not found
  - Column/header mismatch
  - Arrays may be written as serialized strings depending on connector behavior
  - Append-only behavior creates duplicates on reruns
- **Sub-workflow reference:** None.

#### Log Skipped Period (Account A)
- **Type and role:** `set`; constructs a log record for skipped or failed periods.
- **Configuration choices:** Raw JSON output with fields:
  - `status: "skipped"`
  - `reason: "Meta API returned no data or an error for this period"`
  - `account: "Account A"`
  - `since` and `until` read from the loop node
  - `execution_id: {{ $execution.id }}`
  - `timestamp: {{ new Date().toISOString() }}`
- **Key expressions or variables used:**
  - `$('Loop Over Periods (Account A)').item.json.since`
  - `$('Loop Over Periods (Account A)').item.json.until`
  - `$execution.id`
  - `new Date().toISOString()`
- **Input and output connections:**
  - Receives from **Valid Response? (Account A)** false branch
  - Outputs to **Append to Log Sheet (Account A)**
- **Version-specific requirements:** Set node version 3.4.
- **Edge cases or potential failure types:**
  - If loop item context is unavailable, the expressions can fail.
  - Does not preserve the actual Meta error payload, only a generic reason.
- **Sub-workflow reference:** None.

#### Append to Log Sheet (Account A)
- **Type and role:** `googleSheets`; appends skipped-period logs to a dedicated tab.
- **Configuration choices:**
  - Operation: `append`
  - Spreadsheet: `YOUR_SPREADSHEET_ID`
  - Sheet name: `Account_A_Log`
  - Explicit mapping for log fields
- **Key expressions or variables used:** `{{ $json.status }}`, `{{ $json.reason }}`, etc.
- **Input and output connections:**
  - Receives from **Log Skipped Period (Account A)**
  - Outputs back to **Loop Over Periods (Account A)** to continue iteration
- **Version-specific requirements:** Google Sheets node version 4.7.
- **Edge cases or potential failure types:**
  - Same Sheets credential and schema risks as other Sheets nodes
  - Log writes can fail and interrupt loop progression
- **Sub-workflow reference:** None.

---

## 2.5 Account B — Ad-Level Daily Extraction, Validation, Filtering, Load, and Logging

### Overview
This branch processes each period for Account B. It fetches ad-level daily insights, validates the response, expands the result into row items, filters out zero-spend rows, appends valid rows to the `Account_B` sheet, and logs skipped periods to `Account_B_Log`.

### Nodes Involved
- Loop Over Periods (Account B)
- Fetch Meta Insights (Account B)
- Valid Response? (Account B)
- Split API Response (Account B)
- If Spend not 0 (Account B)
- Append to Sheet (Account B)
- Log Skipped Period (Account B)
- Append to Log Sheet (Account B)

### Node Details

#### Loop Over Periods (Account B)
- **Type and role:** `splitInBatches`; iterates through each period item for Account B.
- **Configuration choices:** Default batch iteration behavior.
- **Key expressions or variables used:** Referenced later for log context.
- **Input and output connections:**
  - Receives from **Generate Weekly Periods** and **Set Incremental Range (Last 7 Days)**
  - Output index 1 goes to **Fetch Meta Insights (Account B)**
  - Loop-back from **Append to Sheet (Account B)** and **Append to Log Sheet (Account B)**
- **Version-specific requirements:** Split In Batches version 3.
- **Edge cases or potential failure types:**
  - Same looping risks as Account A branch
- **Sub-workflow reference:** None.

#### Fetch Meta Insights (Account B)
- **Type and role:** `httpRequest`; calls the Meta Insights endpoint for Account B.
- **Configuration choices:**
  - URL: `https://graph.facebook.com/v24.0/act_XXXXXXXXXXXXXXXXX/insights`
  - Authentication: HTTP Header Auth credential
  - Query parameters:
    - same `fields` list as Account A
    - `level=ad`
    - `time_increment=1`
    - `time_range[since]={{ $json.since }}`
    - `time_range[until]={{ $json.until }}`
  - Pagination enabled via `paging.next`
  - `neverError: true`
  - `retryOnFail: true`
  - `maxTries: 5`
  - `waitBetweenTries: 3000`
  - `continueOnFail: true`
- **Key expressions or variables used:**
  - `$json.since`
  - `$json.until`
  - `$response.body.paging.next`
  - `!$response.body.paging?.next`
- **Input and output connections:** Receives from **Loop Over Periods (Account B)**; outputs to **Valid Response? (Account B)**.
- **Version-specific requirements:** HTTP Request node version 4.3.
- **Edge cases or potential failure types:**
  - Same authentication, rate-limit, and error-swallowing risks as Account A
  - Daily ad-level granularity may return much larger datasets, increasing runtime and pagination volume
- **Sub-workflow reference:** None.

#### Valid Response? (Account B)
- **Type and role:** `if`; checks for a non-empty `data` array.
- **Configuration choices:** Same boolean logic as Account A.
- **Key expressions or variables used:**
  - `Array.isArray($json.data) && $json.data.length > 0`
- **Input and output connections:**
  - Receives from **Fetch Meta Insights (Account B)**
  - True output goes to **Split API Response (Account B)**
  - False output goes to **Log Skipped Period (Account B)**
- **Version-specific requirements:** If node version 2.
- **Edge cases or potential failure types:**
  - Valid empty periods and actual API errors are grouped together
- **Sub-workflow reference:** None.

#### Split API Response (Account B)
- **Type and role:** `code`; flattens API response rows into one item per insight row.
- **Configuration choices:** Same mapping and numeric normalization logic as Account A.
- **Key expressions or variables used:** `item.json.data || []`, numeric casts via `Number(...)`.
- **Input and output connections:** Receives from **Valid Response? (Account B)** true branch; outputs to **If Spend not 0 (Account B)**.
- **Version-specific requirements:** Code node version 2.
- **Edge cases or potential failure types:**
  - Same schema drift and array serialization risks as Account A
- **Sub-workflow reference:** None.

#### If Spend not 0 (Account B)
- **Type and role:** `if`; removes zero-spend rows.
- **Configuration choices:**
  - Number `notEquals 0`
  - Loose type validation enabled
- **Key expressions or variables used:** `$json.spend`
- **Input and output connections:**
  - Receives from **Split API Response (Account B)**
  - True output goes to **Append to Sheet (Account B)**
- **Version-specific requirements:** If node version 2.2.
- **Edge cases or potential failure types:**
  - Same loose typing considerations as Account A
- **Sub-workflow reference:** None.

#### Append to Sheet (Account B)
- **Type and role:** `googleSheets`; appends output rows into the Account B reporting tab.
- **Configuration choices:**
  - Operation: `append`
  - Spreadsheet: `YOUR_SPREADSHEET_ID`
  - Sheet: `Account_B` (selected by gid)
  - Explicit field mapping
  - Type conversion disabled
- **Key expressions or variables used:** `{{ $json.fieldName }}`
- **Input and output connections:**
  - Receives from **If Spend not 0 (Account B)** true branch
  - Outputs back to **Loop Over Periods (Account B)**
- **Version-specific requirements:** Google Sheets node version 4.7.
- **Edge cases or potential failure types:**
  - Same Google Sheets credential, schema, and duplicate-row risks as Account A
- **Sub-workflow reference:** None.

#### Log Skipped Period (Account B)
- **Type and role:** `set`; builds a log row for skipped periods.
- **Configuration choices:** Raw JSON output with:
  - `status: "skipped"`
  - `reason: "Meta API returned no data or an error for this period"`
  - `account: "Account B"`
  - loop-derived `since` and `until`
  - execution ID and ISO timestamp
- **Key expressions or variables used:**
  - `$('Loop Over Periods (Account B)').item.json.since`
  - `$('Loop Over Periods (Account B)').item.json.until`
  - `$execution.id`
  - `new Date().toISOString()`
- **Input and output connections:**
  - Receives from **Valid Response? (Account B)** false branch
  - Outputs to **Append to Log Sheet (Account B)**
- **Version-specific requirements:** Set node version 3.4.
- **Edge cases or potential failure types:**
  - Same contextual expression risks as Account A log node
- **Sub-workflow reference:** None.

#### Append to Log Sheet (Account B)
- **Type and role:** `googleSheets`; appends skip/error logs into `Account_B_Log`.
- **Configuration choices:**
  - Operation: `append`
  - Spreadsheet: `YOUR_SPREADSHEET_ID`
  - Sheet name: `Account_B_Log`
  - Explicit mapping for log fields
- **Key expressions or variables used:** `{{ $json.status }}`, etc.
- **Input and output connections:**
  - Receives from **Log Skipped Period (Account B)**
  - Outputs back to **Loop Over Periods (Account B)**
- **Version-specific requirements:** Google Sheets node version 4.7.
- **Edge cases or potential failure types:**
  - Same log-sheet and credential risks as Account A
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Manual Trigger (Historical Backfill) | manualTrigger | Manual entry point for historical backfill runs |  | Set Date Range (Historical Backfill) | ## Meta Ads Insights to Google Sheets (Backfill & Weekly Sync ETL)<br>This workflow extracts Meta Ads performance data and loads it into Google Sheets for reporting or use as a BI source such as Looker Studio. It supports two operating modes in a single workflow.<br>---<br>### Two Paths, One Workflow<br>**Path 1 — Historical Backfill (Manual Trigger)**<br>Run once to populate past data. Set `start_date` and `end_date` in the *Set Date Range* node. The *Generate Weekly Periods* node splits the range into 7-day chunks and processes them one by one to reduce the chance of API throttling.<br>**Path 2 — Incremental Sync (Schedule Trigger, every 7 days)**<br>Runs automatically and pulls the last 7 days. The date range is calculated dynamically, so no manual input is needed.<br>---<br>### Granularity Per Account<br>- **Account A** — `level=campaign`, weekly<br>- **Account B** — `level=ad` + `time_increment=1`, daily per ad<br>---<br>### Known Limitation<br>This template uses append-only writes. Re-running over the same date range will produce duplicate rows. Manual deduplication or a cleanup step may be needed.<br>---<br>### Setup<br>a) **Meta credential** — Configure an HTTP Header Auth credential with a long-lived Meta User Access Token that has `ads_read` permission.<br>b) **Ad Account IDs** — Replace `act_XXXXXXXXXXXXXXXXX` in both HTTP Request nodes.<br>c) **Google Sheets** — Configure a Google Sheets OAuth2 credential and prepare tabs `Account_A`, `Account_B`, `Account_A_Log`, and `Account_B_Log`.<br>d) **Historical range** — Update `start_date` and `end_date` in *Set Date Range* before running the Manual Trigger. |
| Set Date Range (Historical Backfill) | set | Defines static historical backfill date range | Manual Trigger (Historical Backfill) | Generate Weekly Periods | ## Meta Ads Insights to Google Sheets (Backfill & Weekly Sync ETL)<br>This workflow extracts Meta Ads performance data and loads it into Google Sheets for reporting or use as a BI source such as Looker Studio. It supports two operating modes in a single workflow.<br>---<br>### Two Paths, One Workflow<br>**Path 1 — Historical Backfill (Manual Trigger)**<br>Run once to populate past data. Set `start_date` and `end_date` in the *Set Date Range* node. The *Generate Weekly Periods* node splits the range into 7-day chunks and processes them one by one to reduce the chance of API throttling.<br>**Path 2 — Incremental Sync (Schedule Trigger, every 7 days)**<br>Runs automatically and pulls the last 7 days. The date range is calculated dynamically, so no manual input is needed.<br>---<br>### Granularity Per Account<br>- **Account A** — `level=campaign`, weekly<br>- **Account B** — `level=ad` + `time_increment=1`, daily per ad<br>---<br>### Known Limitation<br>This template uses append-only writes. Re-running over the same date range will produce duplicate rows. Manual deduplication or a cleanup step may be needed.<br>---<br>### Setup<br>a) **Meta credential** — Configure an HTTP Header Auth credential with a long-lived Meta User Access Token that has `ads_read` permission.<br>b) **Ad Account IDs** — Replace `act_XXXXXXXXXXXXXXXXX` in both HTTP Request nodes.<br>c) **Google Sheets** — Configure a Google Sheets OAuth2 credential and prepare tabs `Account_A`, `Account_B`, `Account_A_Log`, and `Account_B_Log`.<br>d) **Historical range** — Update `start_date` and `end_date` in *Set Date Range* before running the Manual Trigger. |
| Generate Weekly Periods | code | Splits historical range into 7-day periods | Set Date Range (Historical Backfill) | Loop Over Periods (Account A), Loop Over Periods (Account B) | ## Meta Ads Insights to Google Sheets (Backfill & Weekly Sync ETL)<br>This workflow extracts Meta Ads performance data and loads it into Google Sheets for reporting or use as a BI source such as Looker Studio. It supports two operating modes in a single workflow.<br>---<br>### Two Paths, One Workflow<br>**Path 1 — Historical Backfill (Manual Trigger)**<br>Run once to populate past data. Set `start_date` and `end_date` in the *Set Date Range* node. The *Generate Weekly Periods* node splits the range into 7-day chunks and processes them one by one to reduce the chance of API throttling.<br>**Path 2 — Incremental Sync (Schedule Trigger, every 7 days)**<br>Runs automatically and pulls the last 7 days. The date range is calculated dynamically, so no manual input is needed.<br>---<br>### Granularity Per Account<br>- **Account A** — `level=campaign`, weekly<br>- **Account B** — `level=ad` + `time_increment=1`, daily per ad<br>---<br>### Known Limitation<br>This template uses append-only writes. Re-running over the same date range will produce duplicate rows. Manual deduplication or a cleanup step may be needed.<br>---<br>### Setup<br>a) **Meta credential** — Configure an HTTP Header Auth credential with a long-lived Meta User Access Token that has `ads_read` permission.<br>b) **Ad Account IDs** — Replace `act_XXXXXXXXXXXXXXXXX` in both HTTP Request nodes.<br>c) **Google Sheets** — Configure a Google Sheets OAuth2 credential and prepare tabs `Account_A`, `Account_B`, `Account_A_Log`, and `Account_B_Log`.<br>d) **Historical range** — Update `start_date` and `end_date` in *Set Date Range* before running the Manual Trigger. |
| Schedule Trigger (Incremental) | scheduleTrigger | Scheduled entry point running every 7 days |  | Set Incremental Range (Last 7 Days) | ## Meta Ads Insights to Google Sheets (Backfill & Weekly Sync ETL)<br>This workflow extracts Meta Ads performance data and loads it into Google Sheets for reporting or use as a BI source such as Looker Studio. It supports two operating modes in a single workflow.<br>---<br>### Two Paths, One Workflow<br>**Path 1 — Historical Backfill (Manual Trigger)**<br>Run once to populate past data. Set `start_date` and `end_date` in the *Set Date Range* node. The *Generate Weekly Periods* node splits the range into 7-day chunks and processes them one by one to reduce the chance of API throttling.<br>**Path 2 — Incremental Sync (Schedule Trigger, every 7 days)**<br>Runs automatically and pulls the last 7 days. The date range is calculated dynamically, so no manual input is needed.<br>---<br>### Granularity Per Account<br>- **Account A** — `level=campaign`, weekly<br>- **Account B** — `level=ad` + `time_increment=1`, daily per ad<br>---<br>### Known Limitation<br>This template uses append-only writes. Re-running over the same date range will produce duplicate rows. Manual deduplication or a cleanup step may be needed.<br>---<br>### Setup<br>a) **Meta credential** — Configure an HTTP Header Auth credential with a long-lived Meta User Access Token that has `ads_read` permission.<br>b) **Ad Account IDs** — Replace `act_XXXXXXXXXXXXXXXXX` in both HTTP Request nodes.<br>c) **Google Sheets** — Configure a Google Sheets OAuth2 credential and prepare tabs `Account_A`, `Account_B`, `Account_A_Log`, and `Account_B_Log`.<br>d) **Historical range** — Update `start_date` and `end_date` in *Set Date Range* before running the Manual Trigger. |
| Set Incremental Range (Last 7 Days) | set | Creates rolling 7-day date range | Schedule Trigger (Incremental) | Loop Over Periods (Account A), Loop Over Periods (Account B) | ## Meta Ads Insights to Google Sheets (Backfill & Weekly Sync ETL)<br>This workflow extracts Meta Ads performance data and loads it into Google Sheets for reporting or use as a BI source such as Looker Studio. It supports two operating modes in a single workflow.<br>---<br>### Two Paths, One Workflow<br>**Path 1 — Historical Backfill (Manual Trigger)**<br>Run once to populate past data. Set `start_date` and `end_date` in the *Set Date Range* node. The *Generate Weekly Periods* node splits the range into 7-day chunks and processes them one by one to reduce the chance of API throttling.<br>**Path 2 — Incremental Sync (Schedule Trigger, every 7 days)**<br>Runs automatically and pulls the last 7 days. The date range is calculated dynamically, so no manual input is needed.<br>---<br>### Granularity Per Account<br>- **Account A** — `level=campaign`, weekly<br>- **Account B** — `level=ad` + `time_increment=1`, daily per ad<br>---<br>### Known Limitation<br>This template uses append-only writes. Re-running over the same date range will produce duplicate rows. Manual deduplication or a cleanup step may be needed.<br>---<br>### Setup<br>a) **Meta credential** — Configure an HTTP Header Auth credential with a long-lived Meta User Access Token that has `ads_read` permission.<br>b) **Ad Account IDs** — Replace `act_XXXXXXXXXXXXXXXXX` in both HTTP Request nodes.<br>c) **Google Sheets** — Configure a Google Sheets OAuth2 credential and prepare tabs `Account_A`, `Account_B`, `Account_A_Log`, and `Account_B_Log`.<br>d) **Historical range** — Update `start_date` and `end_date` in *Set Date Range* before running the Manual Trigger. |
| Loop Over Periods (Account A) | splitInBatches | Iterates through Account A date windows | Generate Weekly Periods, Set Incremental Range (Last 7 Days), Append to Sheet (Account A), Append to Log Sheet (Account A) | Fetch Meta Insights (Account A) | ## Ad Account A — Campaign Level<br>Fetches weekly campaign-level insights from the Meta Ads API.<br>**Setup:**<br>- Replace `act_XXXXXXXXXXXXXXXXX` in the HTTP node URL with your Meta Ad Account ID.<br>- Target sheet tab: **Account_A**<br>- Granularity: `level=campaign` (one row per campaign per week). |
| Fetch Meta Insights (Account A) | httpRequest | Calls Meta Ads Insights API for Account A | Loop Over Periods (Account A) | Valid Response? (Account A) | ## Ad Account A — Campaign Level<br>Fetches weekly campaign-level insights from the Meta Ads API.<br>**Setup:**<br>- Replace `act_XXXXXXXXXXXXXXXXX` in the HTTP node URL with your Meta Ad Account ID.<br>- Target sheet tab: **Account_A**<br>- Granularity: `level=campaign` (one row per campaign per week). |
| Valid Response? (Account A) | if | Validates that Account A API response contains rows | Fetch Meta Insights (Account A) | Split API Response (Account A), Log Skipped Period (Account A) | ## Ad Account A — Campaign Level<br>Fetches weekly campaign-level insights from the Meta Ads API.<br>**Setup:**<br>- Replace `act_XXXXXXXXXXXXXXXXX` in the HTTP node URL with your Meta Ad Account ID.<br>- Target sheet tab: **Account_A**<br>- Granularity: `level=campaign` (one row per campaign per week). |
| Split API Response (Account A) | code | Expands Account A API data array into row items | Valid Response? (Account A) | If Spend not 0 (Account A) | ## Ad Account A — Campaign Level<br>Fetches weekly campaign-level insights from the Meta Ads API.<br>**Setup:**<br>- Replace `act_XXXXXXXXXXXXXXXXX` in the HTTP node URL with your Meta Ad Account ID.<br>- Target sheet tab: **Account_A**<br>- Granularity: `level=campaign` (one row per campaign per week). |
| If Spend not 0 (Account A) | if | Filters out zero-spend rows for Account A | Split API Response (Account A) | Append to Sheet (Account A) | ## Ad Account A — Campaign Level<br>Fetches weekly campaign-level insights from the Meta Ads API.<br>**Setup:**<br>- Replace `act_XXXXXXXXXXXXXXXXX` in the HTTP node URL with your Meta Ad Account ID.<br>- Target sheet tab: **Account_A**<br>- Granularity: `level=campaign` (one row per campaign per week). |
| Append to Sheet (Account A) | googleSheets | Appends Account A rows to Google Sheets | If Spend not 0 (Account A) | Loop Over Periods (Account A) | ## Ad Account A — Campaign Level<br>Fetches weekly campaign-level insights from the Meta Ads API.<br>**Setup:**<br>- Replace `act_XXXXXXXXXXXXXXXXX` in the HTTP node URL with your Meta Ad Account ID.<br>- Target sheet tab: **Account_A**<br>- Granularity: `level=campaign` (one row per campaign per week). |
| Log Skipped Period (Account A) | set | Builds skip/error log row for Account A | Valid Response? (Account A) | Append to Log Sheet (Account A) | ## Ad Account A — Campaign Level<br>Fetches weekly campaign-level insights from the Meta Ads API.<br>**Setup:**<br>- Replace `act_XXXXXXXXXXXXXXXXX` in the HTTP node URL with your Meta Ad Account ID.<br>- Target sheet tab: **Account_A**<br>- Granularity: `level=campaign` (one row per campaign per week). |
| Append to Log Sheet (Account A) | googleSheets | Appends Account A skipped-period log to Google Sheets | Log Skipped Period (Account A) | Loop Over Periods (Account A) | ## Ad Account A — Campaign Level<br>Fetches weekly campaign-level insights from the Meta Ads API.<br>**Setup:**<br>- Replace `act_XXXXXXXXXXXXXXXXX` in the HTTP node URL with your Meta Ad Account ID.<br>- Target sheet tab: **Account_A**<br>- Granularity: `level=campaign` (one row per campaign per week). |
| Loop Over Periods (Account B) | splitInBatches | Iterates through Account B date windows | Generate Weekly Periods, Set Incremental Range (Last 7 Days), Append to Sheet (Account B), Append to Log Sheet (Account B) | Fetch Meta Insights (Account B) | ## Ad Account B — Ad Level (Daily Breakdown)<br>Fetches weekly ad-level insights with a daily breakdown from the Meta Ads API.<br>**Setup:**<br>- Replace `act_XXXXXXXXXXXXXXXXX` in the HTTP node URL with your Meta Ad Account ID.<br>- Target sheet tab: **Account_B**<br>- Granularity: `level=ad` + `time_increment=1` (one row per ad per day). |
| Fetch Meta Insights (Account B) | httpRequest | Calls Meta Ads Insights API for Account B at ad/day granularity | Loop Over Periods (Account B) | Valid Response? (Account B) | ## Ad Account B — Ad Level (Daily Breakdown)<br>Fetches weekly ad-level insights with a daily breakdown from the Meta Ads API.<br>**Setup:**<br>- Replace `act_XXXXXXXXXXXXXXXXX` in the HTTP node URL with your Meta Ad Account ID.<br>- Target sheet tab: **Account_B**<br>- Granularity: `level=ad` + `time_increment=1` (one row per ad per day). |
| Valid Response? (Account B) | if | Validates that Account B API response contains rows | Fetch Meta Insights (Account B) | Split API Response (Account B), Log Skipped Period (Account B) | ## Ad Account B — Ad Level (Daily Breakdown)<br>Fetches weekly ad-level insights with a daily breakdown from the Meta Ads API.<br>**Setup:**<br>- Replace `act_XXXXXXXXXXXXXXXXX` in the HTTP node URL with your Meta Ad Account ID.<br>- Target sheet tab: **Account_B**<br>- Granularity: `level=ad` + `time_increment=1` (one row per ad per day). |
| Split API Response (Account B) | code | Expands Account B API data array into row items | Valid Response? (Account B) | If Spend not 0 (Account B) | ## Ad Account B — Ad Level (Daily Breakdown)<br>Fetches weekly ad-level insights with a daily breakdown from the Meta Ads API.<br>**Setup:**<br>- Replace `act_XXXXXXXXXXXXXXXXX` in the HTTP node URL with your Meta Ad Account ID.<br>- Target sheet tab: **Account_B**<br>- Granularity: `level=ad` + `time_increment=1` (one row per ad per day). |
| If Spend not 0 (Account B) | if | Filters out zero-spend rows for Account B | Split API Response (Account B) | Append to Sheet (Account B) | ## Ad Account B — Ad Level (Daily Breakdown)<br>Fetches weekly ad-level insights with a daily breakdown from the Meta Ads API.<br>**Setup:**<br>- Replace `act_XXXXXXXXXXXXXXXXX` in the HTTP node URL with your Meta Ad Account ID.<br>- Target sheet tab: **Account_B**<br>- Granularity: `level=ad` + `time_increment=1` (one row per ad per day). |
| Append to Sheet (Account B) | googleSheets | Appends Account B rows to Google Sheets | If Spend not 0 (Account B) | Loop Over Periods (Account B) | ## Ad Account B — Ad Level (Daily Breakdown)<br>Fetches weekly ad-level insights with a daily breakdown from the Meta Ads API.<br>**Setup:**<br>- Replace `act_XXXXXXXXXXXXXXXXX` in the HTTP node URL with your Meta Ad Account ID.<br>- Target sheet tab: **Account_B**<br>- Granularity: `level=ad` + `time_increment=1` (one row per ad per day). |
| Log Skipped Period (Account B) | set | Builds skip/error log row for Account B | Valid Response? (Account B) | Append to Log Sheet (Account B) | ## Ad Account B — Ad Level (Daily Breakdown)<br>Fetches weekly ad-level insights with a daily breakdown from the Meta Ads API.<br>**Setup:**<br>- Replace `act_XXXXXXXXXXXXXXXXX` in the HTTP node URL with your Meta Ad Account ID.<br>- Target sheet tab: **Account_B**<br>- Granularity: `level=ad` + `time_increment=1` (one row per ad per day). |
| Append to Log Sheet (Account B) | googleSheets | Appends Account B skipped-period log to Google Sheets | Log Skipped Period (Account B) | Loop Over Periods (Account B) | ## Ad Account B — Ad Level (Daily Breakdown)<br>Fetches weekly ad-level insights with a daily breakdown from the Meta Ads API.<br>**Setup:**<br>- Replace `act_XXXXXXXXXXXXXXXXX` in the HTTP node URL with your Meta Ad Account ID.<br>- Target sheet tab: **Account_B**<br>- Granularity: `level=ad` + `time_increment=1` (one row per ad per day). |
| Workflow Overview | stickyNote | Documentation note |  |  |  |
| Sticky Note3 | stickyNote | Documentation note for Account A branch |  |  |  |
| Sticky Note4 | stickyNote | Documentation note for Account B branch |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
   - Name it something like: `Meta Ads Insights to Google Sheets (Backfill & Weekly Sync ETL)`.

2. **Create the Meta credential.**
   - Add a credential of type **HTTP Header Auth**.
   - Configure it to send an authorization header suitable for Meta Graph API.
   - Typically this is a bearer token header using a long-lived Meta User Access Token with `ads_read` permission.
   - Save the credential with a recognizable name.

3. **Create the Google Sheets credential.**
   - Add a **Google Sheets OAuth2** credential.
   - Authenticate it against the Google account that owns or can edit the target spreadsheet.
   - Save the credential.

4. **Prepare the target Google Sheets file.**
   - Create one spreadsheet, for example `Meta Ads Report`.
   - Create four tabs:
     - `Account_A`
     - `Account_B`
     - `Account_A_Log`
     - `Account_B_Log`

5. **Add the Manual Trigger node.**
   - Node type: **Manual Trigger**
   - Name it: `Manual Trigger (Historical Backfill)`

6. **Add a Set node for backfill dates.**
   - Node type: **Set**
   - Name it: `Set Date Range (Historical Backfill)`
   - Use **raw JSON output** mode.
   - Set:
     - `start_date`: `2024-01-01`
     - `end_date`: `2024-12-31`
   - Connect:
     - `Manual Trigger (Historical Backfill)` → `Set Date Range (Historical Backfill)`

7. **Add a Code node to generate weekly periods.**
   - Node type: **Code**
   - Name it: `Generate Weekly Periods`
   - Paste logic equivalent to:
     - read `start_date` and `end_date`
     - create chunks of 7 days
     - clamp last chunk to the end date
     - return one item per `{ since, until }`
   - Connect:
     - `Set Date Range (Historical Backfill)` → `Generate Weekly Periods`

8. **Add the scheduled trigger.**
   - Node type: **Schedule Trigger**
   - Name it: `Schedule Trigger (Incremental)`
   - Configure interval rule:
     - every **7 days**

9. **Add a Set node for the incremental range.**
   - Node type: **Set**
   - Name it: `Set Incremental Range (Last 7 Days)`
   - Use **raw JSON output** mode.
   - Set:
     - `since = {{ $now.minus(7, 'days').toISODate() }}`
     - `until = {{ $now.toISODate() }}`
   - Connect:
     - `Schedule Trigger (Incremental)` → `Set Incremental Range (Last 7 Days)`

10. **Add the Account A loop node.**
    - Node type: **Split In Batches**
    - Name it: `Loop Over Periods (Account A)`
    - Keep default options.
    - Connect both date sources into it:
      - `Generate Weekly Periods` → `Loop Over Periods (Account A)`
      - `Set Incremental Range (Last 7 Days)` → `Loop Over Periods (Account A)`

11. **Add the Account A HTTP Request node.**
    - Node type: **HTTP Request**
    - Name it: `Fetch Meta Insights (Account A)`
    - Method: `GET`
    - URL:
      - `https://graph.facebook.com/v24.0/act_XXXXXXXXXXXXXXXXX/insights`
    - Replace `act_XXXXXXXXXXXXXXXXX` with the real Meta ad account ID for Account A.
    - Authentication:
      - Generic Credential Type
      - HTTP Header Auth
      - Select the Meta credential
    - Enable query parameters and add:
      - `fields = date_start,date_stop,account_id,account_name,campaign_id,campaign_name,adset_id,adset_name,ad_id,ad_name,objective,buying_type,spend,impressions,reach,frequency,clicks,unique_clicks,ctr,cpc,cpm,actions,cost_per_action_type,action_values,quality_ranking,engagement_rate_ranking,conversion_rate_ranking`
      - `level = campaign`
      - `time_range[since] = {{ $json.since }}`
      - `time_range[until] = {{ $json.until }}`
    - Configure options:
      - response should **never error**
      - enable pagination using next URL from `{{ $response.body.paging.next }}`
      - complete when `{{ !$response.body.paging?.next }}`
      - request interval: `3000 ms`
    - Enable retries:
      - `retryOnFail = true`
      - `maxTries = 5`
      - `waitBetweenTries = 3000`
      - `continueOnFail = true`
    - Connect:
      - `Loop Over Periods (Account A)` second/main iteration output → `Fetch Meta Insights (Account A)`

12. **Add the Account A response validation node.**
    - Node type: **If**
    - Name it: `Valid Response? (Account A)`
    - Condition:
      - boolean equals `true`
      - left value: `{{ Array.isArray($json.data) && $json.data.length > 0 }}`
    - Connect:
      - `Fetch Meta Insights (Account A)` → `Valid Response? (Account A)`

13. **Add the Account A response splitter.**
    - Node type: **Code**
    - Name it: `Split API Response (Account A)`
    - Use logic that:
      - iterates through `item.json.data`
      - outputs one item per row
      - copies all metadata fields
      - converts spend/impressions/reach/frequency/clicks/unique_clicks/ctr/cpc/cpm to numbers
      - defaults arrays to `[]`
      - defaults ranking strings to empty string
    - Connect:
      - true output of `Valid Response? (Account A)` → `Split API Response (Account A)`

14. **Add the Account A spend filter.**
    - Node type: **If**
    - Name it: `If Spend not 0 (Account A)`
    - Condition:
      - number `not equals`
      - left value: `{{ $json.spend }}`
      - right value: `0`
    - Enable loose type validation.
    - Connect:
      - `Split API Response (Account A)` → `If Spend not 0 (Account A)`

15. **Add the Account A Google Sheets append node.**
    - Node type: **Google Sheets**
    - Name it: `Append to Sheet (Account A)`
    - Operation: **Append**
    - Select the Google Sheets credential.
    - Spreadsheet: your target spreadsheet ID.
    - Sheet/tab: `Account_A`
    - Use explicit column mapping for:
      - `date_start`
      - `date_stop`
      - `account_id`
      - `account_name`
      - `campaign_id`
      - `campaign_name`
      - `adset_id`
      - `adset_name`
      - `ad_id`
      - `ad_name`
      - `objective`
      - `buying_type`
      - `spend`
      - `impressions`
      - `reach`
      - `frequency`
      - `clicks`
      - `unique_clicks`
      - `ctr`
      - `cpc`
      - `cpm`
      - `actions`
      - `cost_per_action_type`
      - `action_values`
      - `quality_ranking`
      - `engagement_rate_ranking`
      - `conversion_rate_ranking`
    - Use expressions like `{{ $json.field_name }}`.
    - Connect:
      - true output of `If Spend not 0 (Account A)` → `Append to Sheet (Account A)`

16. **Add the Account A skip log Set node.**
    - Node type: **Set**
    - Name it: `Log Skipped Period (Account A)`
    - Use raw JSON output:
      - `status = "skipped"`
      - `reason = "Meta API returned no data or an error for this period"`
      - `account = "Account A"`
      - `since = {{ $('Loop Over Periods (Account A)').item.json.since }}`
      - `until = {{ $('Loop Over Periods (Account A)').item.json.until }}`
      - `execution_id = {{ $execution.id }}`
      - `timestamp = {{ new Date().toISOString() }}`
    - Connect:
      - false output of `Valid Response? (Account A)` → `Log Skipped Period (Account A)`

17. **Add the Account A log sheet node.**
    - Node type: **Google Sheets**
    - Name it: `Append to Log Sheet (Account A)`
    - Operation: **Append**
    - Spreadsheet: same spreadsheet
    - Sheet/tab: `Account_A_Log`
    - Map:
      - `status`
      - `reason`
      - `account`
      - `since`
      - `until`
      - `execution_id`
      - `timestamp`
    - Connect:
      - `Log Skipped Period (Account A)` → `Append to Log Sheet (Account A)`

18. **Close the Account A loop.**
    - Connect:
      - `Append to Sheet (Account A)` → `Loop Over Periods (Account A)`
      - `Append to Log Sheet (Account A)` → `Loop Over Periods (Account A)`

19. **Add the Account B loop node.**
    - Node type: **Split In Batches**
    - Name it: `Loop Over Periods (Account B)`
    - Connect both date sources:
      - `Generate Weekly Periods` → `Loop Over Periods (Account B)`
      - `Set Incremental Range (Last 7 Days)` → `Loop Over Periods (Account B)`

20. **Add the Account B HTTP Request node.**
    - Node type: **HTTP Request**
    - Name it: `Fetch Meta Insights (Account B)`
    - Method: `GET`
    - URL:
      - `https://graph.facebook.com/v24.0/act_XXXXXXXXXXXXXXXXX/insights`
    - Replace with the real ad account ID for Account B.
    - Authentication:
      - Generic Credential Type
      - HTTP Header Auth
      - Select the Meta credential
    - Query parameters:
      - same `fields` list as Account A
      - `level = ad`
      - `time_increment = 1`
      - `time_range[since] = {{ $json.since }}`
      - `time_range[until] = {{ $json.until }}`
    - Options:
      - `neverError = true`
      - pagination via `{{ $response.body.paging.next }}`
      - complete when `{{ !$response.body.paging?.next }}`
      - request interval `3000 ms`
    - Retry settings:
      - `retryOnFail = true`
      - `maxTries = 5`
      - `waitBetweenTries = 3000`
      - `continueOnFail = true`
    - Connect:
      - `Loop Over Periods (Account B)` second/main iteration output → `Fetch Meta Insights (Account B)`

21. **Add the Account B response validation node.**
    - Node type: **If**
    - Name it: `Valid Response? (Account B)`
    - Condition:
      - `{{ Array.isArray($json.data) && $json.data.length > 0 }}`
      - equals `true`
    - Connect:
      - `Fetch Meta Insights (Account B)` → `Valid Response? (Account B)`

22. **Add the Account B response splitter.**
    - Node type: **Code**
    - Name it: `Split API Response (Account B)`
    - Use the same transformation logic as Account A.
    - Connect:
      - true output of `Valid Response? (Account B)` → `Split API Response (Account B)`

23. **Add the Account B spend filter.**
    - Node type: **If**
    - Name it: `If Spend not 0 (Account B)`
    - Condition:
      - `{{ $json.spend }} != 0`
    - Enable loose type validation.
    - Connect:
      - `Split API Response (Account B)` → `If Spend not 0 (Account B)`

24. **Add the Account B Google Sheets append node.**
    - Node type: **Google Sheets**
    - Name it: `Append to Sheet (Account B)`
    - Operation: **Append**
    - Spreadsheet: same spreadsheet
    - Sheet/tab: `Account_B`
    - Use the same mapped columns as Account A.
    - Connect:
      - true output of `If Spend not 0 (Account B)` → `Append to Sheet (Account B)`

25. **Add the Account B skip log Set node.**
    - Node type: **Set**
    - Name it: `Log Skipped Period (Account B)`
    - Use raw JSON output:
      - `status = "skipped"`
      - `reason = "Meta API returned no data or an error for this period"`
      - `account = "Account B"`
      - `since = {{ $('Loop Over Periods (Account B)').item.json.since }}`
      - `until = {{ $('Loop Over Periods (Account B)').item.json.until }}`
      - `execution_id = {{ $execution.id }}`
      - `timestamp = {{ new Date().toISOString() }}`
    - Connect:
      - false output of `Valid Response? (Account B)` → `Log Skipped Period (Account B)`

26. **Add the Account B log sheet node.**
    - Node type: **Google Sheets**
    - Name it: `Append to Log Sheet (Account B)`
    - Operation: **Append**
    - Spreadsheet: same spreadsheet
    - Sheet/tab: `Account_B_Log`
    - Map:
      - `status`
      - `reason`
      - `account`
      - `since`
      - `until`
      - `execution_id`
      - `timestamp`
    - Connect:
      - `Log Skipped Period (Account B)` → `Append to Log Sheet (Account B)`

27. **Close the Account B loop.**
    - Connect:
      - `Append to Sheet (Account B)` → `Loop Over Periods (Account B)`
      - `Append to Log Sheet (Account B)` → `Loop Over Periods (Account B)`

28. **Optionally add the same sticky notes for maintainability.**
    - One global note describing:
      - workflow purpose
      - two execution paths
      - setup requirements
      - duplicate-row limitation
    - One note above Account A branch
    - One note above Account B branch

29. **Test the historical path first.**
    - Set a small date range in `Set Date Range (Historical Backfill)`, such as one week.
    - Run the manual trigger.
    - Confirm rows appear in:
      - `Account_A`
      - `Account_B`
    - Confirm any empty periods appear in:
      - `Account_A_Log`
      - `Account_B_Log`

30. **Validate pagination and data volume.**
    - Account B can produce many more rows because it is ad-level and daily.
    - If performance is poor, consider:
      - smaller chunks
      - longer request interval
      - additional filtering fields
      - deduplication logic

31. **Activate the workflow only after testing.**
    - Ensure the schedule trigger is enabled by activating the workflow.
    - Confirm server timezone matches reporting expectations.

32. **Record the operational constraints.**
    - Re-running the same period appends duplicates.
    - API errors may be logged as “skipped” because the HTTP node is configured not to hard-fail.
    - Arrays like `actions`, `cost_per_action_type`, and `action_values` may land in Sheets as serialized structures rather than flattened columns.

### Sub-workflow setup
This workflow does **not** use any Execute Workflow / sub-workflow nodes. No sub-workflow configuration is required.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow supports two modes in one design: manual historical backfill and automatic 7-day incremental sync. | Operational design |
| Account A uses campaign-level insights aggregated by week. | Meta Ads API usage |
| Account B uses ad-level insights with `time_increment=1`, producing one row per ad per day. | Meta Ads API usage |
| The workflow writes in append-only mode, so reruns on the same range will create duplicate rows unless deduplication is added. | Known limitation |
| Required setup includes a Meta HTTP Header Auth credential with a long-lived token and `ads_read` permission. | Authentication |
| Required Google Sheets tabs are `Account_A`, `Account_B`, `Account_A_Log`, and `Account_B_Log`. | Data destination setup |