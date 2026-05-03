Send seasonal WooCommerce sales trend reports to Slack and Google Sheets

https://n8nworkflows.xyz/workflows/send-seasonal-woocommerce-sales-trend-reports-to-slack-and-google-sheets-12371


# Send seasonal WooCommerce sales trend reports to Slack and Google Sheets

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow generates a **weekly seasonality/trend report** from **WooCommerce orders**, comparing the most recent 7 days against the **same-length window** one month ago and one year ago. It computes revenue, order count, average order value (AOV), and the top-selling SKU (by quantity), then **posts a formatted summary to Slack** and **appends a log row to Google Sheets**.

**Target use cases:**
- Weekly sales monitoring for ecommerce teams
- Seasonality planning (month-over-month and year-over-year comparisons)
- Lightweight KPI logging for auditing and dashboards (Google Sheets)

### 1.1 Initialization (Trigger + Global variables)
Runs weekly on a schedule, defines configurable parameters (threshold, sheet ID, webhook placeholder), and prepares date ranges.

### 1.2 Data Ingestion (WooCommerce order retrieval)
Fetches completed orders for the current week, then fetches the same-duration periods for last month and last year.

### 1.3 Analysis & Output (Aggregation + Slack + Sheets)
Aggregates KPIs, computes trends with a configurable threshold, then sends a Slack message and logs to Google Sheets.

---

## 2. Block-by-Block Analysis

### Block 1 — Initialization (Schedule + configuration + date windows)

**Overview:**  
Triggers every Monday morning, sets shared configuration values, and calculates the ISO date ranges used by WooCommerce queries.

**Nodes involved:**
- Weekly Trigger
- Global Configuration
- Calculate Date Ranges

#### Node: Weekly Trigger
- **Type / role:** `Schedule Trigger` — workflow entry point (time-based).
- **Configuration (interpreted):**
  - Runs **every week**, on **Monday** (`triggerAtDay: 1`), at **09:00**.
- **Inputs/Outputs:**
  - **No inputs** (trigger).
  - Outputs to **Global Configuration**.
- **Edge cases / failures:**
  - Timezone behavior depends on n8n instance settings; verify your instance timezone if “Monday 9:00” must be local time.

#### Node: Global Configuration
- **Type / role:** `Set` — defines global variables used downstream.
- **Configuration (interpreted):**
  - Sets:
    - `trend_threshold` (number): **5** (percentage points used to classify Increase/Decrease vs Flat)
    - `teams_webhook_url` (string): placeholder URL (note: not used elsewhere in this workflow)
    - `google_sheet_id` (string): placeholder to be replaced with a Google Sheet URL or ID
- **Key variables/expressions:**
  - Downstream code references: `$('Global Configuration').first().json.trend_threshold`
  - Sheets node references: `$('Global Configuration').first().json.google_sheet_id`
- **Inputs/Outputs:**
  - Input from **Weekly Trigger**
  - Output to **Calculate Date Ranges**
- **Edge cases / failures:**
  - If `google_sheet_id` is not set correctly (invalid URL/ID), Google Sheets append will fail.
  - The sticky note says `slack_webhook_url`, but the node actually defines `teams_webhook_url` and Slack uses OAuth credentials (not webhook). This mismatch can confuse operators.

#### Node: Calculate Date Ranges
- **Type / role:** `Code` — generates ISO time windows for data pulls.
- **Configuration (interpreted):**
  - Computes `current`, `lastMonth`, `lastQuarter`, `lastYear` ranges, each with:
    - `after`: start timestamp (ISO)
    - `before`: end timestamp (ISO)
  - Uses a fixed “one week” duration: `7 * 24 * 60 * 60 * 1000`.
  - “Last month/quarter/year” end is computed by subtracting 1/3/12 months/years from *now* and then taking a week prior.
- **Key outputs (JSON):**
  - `current.after`, `current.before`
  - `lastMonth.after`, `lastMonth.before`
  - `lastQuarter.after`, `lastQuarter.before` (note: calculated but not used later)
  - `lastYear.after`, `lastYear.before`
- **Inputs/Outputs:**
  - Input from **Global Configuration**
  - Output to **Fetch Current Week Sales**
- **Edge cases / failures:**
  - Month subtraction with `setMonth()` can behave unexpectedly around month-end (e.g., March 31 → subtract 1 month). This can shift dates.
  - ISO timestamps are in UTC; WooCommerce filtering may interpret timezone differently depending on store settings/API behavior.

---

### Block 2 — Data Ingestion (WooCommerce)

**Overview:**  
Sequentially fetches orders from WooCommerce for each time window to ensure all datasets exist before analysis runs.

**Nodes involved:**
- Fetch Current Week Sales
- Fetch Last Month Sales
- Fetch Last Year Sales

#### Node: Fetch Current Week Sales
- **Type / role:** `WooCommerce` — retrieves orders for the current 7-day window.
- **Configuration (interpreted):**
  - Resource: **Order**
  - Operation: **Get All**
  - Return: **All items**
  - Filters (options):
    - `after`: from `Calculate Date Ranges.current.after`
    - `before`: from `Calculate Date Ranges.current.before`
    - `status`: `completed`
- **Key expressions:**
  - `after = {{ $('Calculate Date Ranges').first().json.current.after }}`
  - `before = {{ $('Calculate Date Ranges').first().json.current.before }}`
- **Credentials:**
  - Uses `WooCommerce account 3` (WooCommerce API credentials).
- **Inputs/Outputs:**
  - Input from **Calculate Date Ranges**
  - Output to **Fetch Last Month Sales**
- **Edge cases / failures:**
  - Auth/permissions errors (invalid consumer key/secret).
  - Large order volumes can trigger pagination/rate limits; `returnAll: true` may be slow.
  - Status is enforced only here; other periods do not specify status (see below).

#### Node: Fetch Last Month Sales
- **Type / role:** `WooCommerce` — retrieves orders for the “same-length week” one month ago.
- **Configuration (interpreted):**
  - Resource: **Order**
  - Operation: **Get All**
  - Return: **All items**
  - Filters:
    - `after`: `lastMonth.after`
    - `before`: `lastMonth.before`
  - **No status filter configured** (unlike current week).
- **Key expressions:**
  - `after = {{ $('Calculate Date Ranges').first().json.lastMonth.after }}`
  - `before = {{ $('Calculate Date Ranges').first().json.lastMonth.before }}`
- **Credentials:**
  - `WooCommerce account 3`
- **Inputs/Outputs:**
  - Input from **Fetch Current Week Sales**
  - Output to **Fetch Last Year Sales**
- **Edge cases / failures:**
  - Since status is not filtered, results may include `pending`, `cancelled`, etc., depending on WooCommerce defaults—this can skew comparisons vs current week (completed-only).
  - Same performance considerations as above.

#### Node: Fetch Last Year Sales
- **Type / role:** `WooCommerce` — retrieves orders for the “same-length week” one year ago.
- **Configuration (interpreted):**
  - Resource: **Order**
  - Operation: **Get All**
  - Return: **All items**
  - Filters:
    - `after`: `lastYear.after`
    - `before`: `lastYear.before`
  - **No status filter configured**.
- **Special setting:**
  - `alwaysOutputData: true` — even if no items are returned, the node will still output something, helping downstream nodes not fail due to missing input.
- **Key expressions:**
  - `after = {{ $('Calculate Date Ranges').first().json.lastYear.after }}`
  - `before = {{ $('Calculate Date Ranges').first().json.lastYear.before }}`
- **Credentials:**
  - `WooCommerce account 3`
- **Inputs/Outputs:**
  - Input from **Fetch Last Month Sales**
  - Output to **Generate Trend Report**
- **Edge cases / failures:**
  - Same as other Woo nodes; additionally:
  - If WooCommerce returns zero orders, ensure downstream code correctly interprets empty `.all()` (it does, by mapping to `[]`).

---

### Block 3 — Analysis & Output (KPI aggregation, Slack, Google Sheets)

**Overview:**  
Aggregates revenue/orders/SKU quantities, determines trend direction using the configured threshold, then sends a formatted Slack message and appends a record to Google Sheets.

**Nodes involved:**
- Generate Trend Report
- Slack Message
- Log to Google Sheets

#### Node: Generate Trend Report
- **Type / role:** `Code` — transforms raw orders into KPI summary + trend classifications.
- **Configuration (interpreted):**
  - Reads:
    - `trend_threshold` from **Global Configuration** (default 5 if missing)
    - order lists from the three Woo nodes via `.all().map(i => i.json)`
  - Computes metrics per period:
    - `revenue`: sum of `order.total` (parsed float)
    - `orders`: count of orders
    - `aov`: revenue / orders (computed but **not returned in final output**)
    - `skus`: quantities grouped by `line_items[].name` (uses item name, not SKU field)
  - Identifies top SKU by highest quantity (actually “top product name”).
  - Trend function:
    - If historical = 0 and current > 0 → percent: `N/A`, trend: `New`
    - Else percent difference vs historical:
      - `Increase` if diff > threshold
      - `Decrease` if diff < -threshold
      - else `Flat`
- **Key variables/expressions:**
  - `const threshold = $('Global Configuration').first().json.trend_threshold || 5;`
  - Accessing upstream nodes by name (hard dependency on node names).
- **Outputs (JSON):**
  - `revenue.current` (string with 2 decimals)
  - `revenue.vsMonth.percent`, `revenue.vsMonth.trend`
  - `revenue.vsYear.percent`, `revenue.vsYear.trend`
  - `orders.current` (number)
  - `orders.vsMonth.*`, `orders.vsYear.*`
  - `topSku` (string)
- **Inputs/Outputs:**
  - Input from **Fetch Last Year Sales** (but internally reads all three Woo nodes by name)
  - Outputs to **Slack Message** and **Log to Google Sheets** (fan-out)
- **Edge cases / failures:**
  - If node names change, the `$('<Node Name>')` references break.
  - `order.total` may be missing or non-numeric in some WooCommerce configurations; code falls back to 0.
  - Uses `line_items[].name` rather than SKU (`sku` field). If multiple variants share a name, counts merge.

#### Node: Slack Message
- **Type / role:** `Slack` — posts formatted KPI summary to a channel.
- **Configuration (interpreted):**
  - Operation: send a message to a selected channel (`select: channel`)
  - Channel: ID `C0A45F2SK51` (cached name: `n8n-message`)
  - Message body uses Slack “mrkdwn”-style formatting and inserts KPI fields from `$json` (output of Generate Trend Report).
  - `includeLinkToWorkflow: false`
- **Credentials:**
  - `Slack account 30` (Slack API / OAuth).
- **Inputs/Outputs:**
  - Input from **Generate Trend Report**
  - No outgoing connections.
- **Edge cases / failures:**
  - Missing permissions (chat:write) or channel access can cause posting errors.
  - If the channel ID changes (workspace migration), message delivery fails.
  - Formatting assumes all referenced fields exist; if code output changes, message may contain blanks.

#### Node: Log to Google Sheets
- **Type / role:** `Google Sheets` — appends a row for auditing/history.
- **Configuration (interpreted):**
  - Operation: **Append**
  - Document: from `Global Configuration.google_sheet_id` (URL mode)
  - Sheet name: `Sheet1`
- **Important behavior note:**
  - No explicit column mapping is shown in the JSON excerpt. In n8n, append typically requires either:
    - defined columns/fields mapping, or
    - a sheet with headers that match incoming JSON keys (depending on node configuration and n8n version).
  - As provided, this node may append a single JSON blob or fail if not fully configured in the UI.
- **Inputs/Outputs:**
  - Input from **Generate Trend Report**
  - No outgoing connections.
- **Edge cases / failures:**
  - Auth errors (Google OAuth not connected).
  - Wrong sheet name (`Sheet1` not present).
  - Invalid document ID/URL in `google_sheet_id`.
  - Column mismatch or missing mapping can prevent append.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Weekly Trigger | Schedule Trigger | Weekly workflow entry point | — | Global Configuration | ## 1. Initialization  \n\n**Trigger & Config**\nSets the weekly schedule and defines global variables (Webhooks, Thresholds). \n**Date Calculation**\nDynamically generates ISO date ranges for Current, Last Month and Last Year periods. |
| Global Configuration | Set | Stores threshold + external IDs | Weekly Trigger | Calculate Date Ranges | ## 1. Initialization  \n\n**Trigger & Config**\nSets the weekly schedule and defines global variables (Webhooks, Thresholds). \n**Date Calculation**\nDynamically generates ISO date ranges for Current, Last Month and Last Year periods. |
| Calculate Date Ranges | Code | Builds ISO date windows | Global Configuration | Fetch Current Week Sales | ## 1. Initialization  \n\n**Trigger & Config**\nSets the weekly schedule and defines global variables (Webhooks, Thresholds). \n**Date Calculation**\nDynamically generates ISO date ranges for Current, Last Month and Last Year periods. |
| Fetch Current Week Sales | WooCommerce | Pull current week completed orders | Calculate Date Ranges | Fetch Last Month Sales | ## 2. Data Ingestion  \n\n**Sequential Fetching**\nRetrives order data one period at a time (Current -> Month -> Year). This sequential chaining ensures all data is available before the analysis step runs, preventing execution errors. |
| Fetch Last Month Sales | WooCommerce | Pull last month comparable window orders | Fetch Current Week Sales | Fetch Last Year Sales | ## 2. Data Ingestion  \n\n**Sequential Fetching**\nRetrives order data one period at a time (Current -> Month -> Year). This sequential chaining ensures all data is available before the analysis step runs, preventing execution errors. |
| Fetch Last Year Sales | WooCommerce | Pull last year comparable window orders | Fetch Last Month Sales | Generate Trend Report | ## 2. Data Ingestion  \n\n**Sequential Fetching**\nRetrives order data one period at a time (Current -> Month -> Year). This sequential chaining ensures all data is available before the analysis step runs, preventing execution errors. |
| Generate Trend Report | Code | Aggregate KPIs + compute trends | Fetch Last Year Sales | Log to Google Sheets; Slack Message | ## 3. Analysis & Output  \n\n**Trend Analysis**\nCompares revenue/orders against history using the defined threshold. Outputs text-based trends.\n**Reporting**\nSends analystic Data to Slack and appends a row to Google Sheets. |
| Slack Message | Slack | Send formatted trend report to Slack channel | Generate Trend Report | — | ## 3. Analysis & Output  \n\n**Trend Analysis**\nCompares revenue/orders against history using the defined threshold. Outputs text-based trends.\n**Reporting**\nSends analystic Data to Slack and appends a row to Google Sheets. |
| Log to Google Sheets | Google Sheets | Append KPI snapshot to a sheet | Generate Trend Report | — | ## 3. Analysis & Output  \n\n**Trend Analysis**\nCompares revenue/orders against history using the defined threshold. Outputs text-based trends.\n**Reporting**\nSends analystic Data to Slack and appends a row to Google Sheets. |
| Overview | Sticky Note | Documentation | — | — |  |
| Sticky Note | Sticky Note | Documentation for block 1 | — | — |  |
| Sticky Note1 | Sticky Note | Documentation for block 2 | — | — |  |
| Sticky Note2 | Sticky Note | Documentation for block 3 | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it similar to: “WooCommerce Seasonal Sales Planning & Monitor → … → Slack”
- Ensure workflow timezone matches your reporting expectations.

2) **Add node: Schedule Trigger**
- Node type: **Schedule Trigger**
- Configure:
  - Interval: **Weeks**
  - Day of week: **Monday**
  - Time: **09:00**
- This is the entry node.

3) **Add node: Set** (name it **Global Configuration**)
- Add fields:
  - `trend_threshold` (Number) = `5`
  - `google_sheet_id` (String) = your Google Sheet **URL** or **ID**
  - (Optional/legacy) `teams_webhook_url` (String) = not used by this workflow; remove or ignore
- Connect: **Schedule Trigger → Global Configuration**

4) **Add node: Code** (name it **Calculate Date Ranges**)  
- Paste logic that outputs:
  - `current.after/before`
  - `lastMonth.after/before`
  - `lastYear.after/before`
  - (Optional) `lastQuarter.after/before` if you plan to extend
- Connect: **Global Configuration → Calculate Date Ranges**

5) **Add node: WooCommerce** (name it **Fetch Current Week Sales**)
- Resource: **Order**
- Operation: **Get All**
- Return All: **true**
- Options:
  - `after` = expression: `$('Calculate Date Ranges').first().json.current.after`
  - `before` = expression: `$('Calculate Date Ranges').first().json.current.before`
  - `status` = `completed`
- Credentials:
  - Configure WooCommerce API credentials (Consumer Key/Secret + store URL) and select them here.
- Connect: **Calculate Date Ranges → Fetch Current Week Sales**

6) **Add node: WooCommerce** (name it **Fetch Last Month Sales**)
- Same settings as above, but:
  - `after` = `$('Calculate Date Ranges').first().json.lastMonth.after`
  - `before` = `$('Calculate Date Ranges').first().json.lastMonth.before`
  - Decide whether to also set `status=completed` to make comparisons consistent.
- Connect: **Fetch Current Week Sales → Fetch Last Month Sales**

7) **Add node: WooCommerce** (name it **Fetch Last Year Sales**)
- Same settings, but:
  - `after` = `$('Calculate Date Ranges').first().json.lastYear.after`
  - `before` = `$('Calculate Date Ranges').first().json.lastYear.before`
  - Consider adding `status=completed` for consistency.
- Enable setting (in node options): **Always Output Data** (so downstream code runs even when no orders).
- Connect: **Fetch Last Month Sales → Fetch Last Year Sales**

8) **Add node: Code** (name it **Generate Trend Report**)
- Implement:
  - Read `trend_threshold` from Global Configuration
  - Load orders from the three Woo nodes (by node name) and compute:
    - revenue, orders, AOV (optional), top SKU/product name
    - trends vs last month and last year with threshold classification
- Output a single JSON object containing:
  - `revenue.current`, `revenue.vsMonth`, `revenue.vsYear`
  - `orders.current`, `orders.vsMonth`, `orders.vsYear`
  - `topSku`
- Connect: **Fetch Last Year Sales → Generate Trend Report**

9) **Add node: Slack** (name it **Slack Message**)
- Resource/Operation: send message (standard Slack “post message” action in n8n)
- Select: **Channel**, choose the target channel
- Text: use expressions pulling from `$json` (the code output), e.g.:
  - `{{ $json.revenue.current }}`
  - `{{ $json.revenue.vsMonth.percent }}`
  - `{{ $json.topSku }}`
- Credentials:
  - Connect Slack API (OAuth). Ensure scopes include **chat:write** and channel access.
- Connect: **Generate Trend Report → Slack Message**

10) **Add node: Google Sheets** (name it **Log to Google Sheets**)
- Operation: **Append**
- Document ID: expression from config:
  - `$('Global Configuration').first().json.google_sheet_id`
- Sheet name: `Sheet1` (or your actual tab name)
- Configure the **columns mapping** explicitly, for example:
  - `timestamp` (set via expression `{{$now}}` or from date node)
  - `revenue_current`, `revenue_vsMonth_percent`, `revenue_vsMonth_trend`, etc.
- Credentials:
  - Authenticate with Google (OAuth) and ensure the account has edit access to the sheet.
- Connect: **Generate Trend Report → Log to Google Sheets**

11) **(Optional) Add Sticky Notes**
- Add notes for “Initialization”, “Data Ingestion”, “Analysis & Output”, and a general “Overview” to match the provided documentation.

12) **Test and activate**
- Execute manually once to confirm:
  - WooCommerce returns expected orders for each window
  - Slack message renders correctly
  - Google Sheets row appends with correct columns
- Activate workflow.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “WooCommerce Seasonality Tracker… triggers every Monday, calculates date ranges and fetches order data… aggregates revenue and orders, identifies the top-selling SKU and determines trends… sent to Slack and logged to Google Sheets.” | From the workflow “Overview” sticky note (in-canvas documentation). |
| Setup note says to configure `slack_webhook_url`, but the workflow actually posts via the **Slack node with OAuth credentials**; also the Set node contains `teams_webhook_url` which is unused. | Potential configuration mismatch to correct for clarity. |
| The workflow computes `lastQuarter` date ranges but does not fetch “last quarter” sales or include it in the report. | Extension opportunity: add a WooCommerce fetch + comparison. |
| For consistent comparisons, consider applying the same WooCommerce `status` filter (e.g., `completed`) to all periods. | Prevents “current completed-only” vs “historical all statuses” skew. |