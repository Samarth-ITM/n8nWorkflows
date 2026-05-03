Send a daily WooCommerce sales snapshot to Slack with Google Sheets logging

https://n8nworkflows.xyz/workflows/send-a-daily-woocommerce-sales-snapshot-to-slack-with-google-sheets-logging-12901


# Send a daily WooCommerce sales snapshot to Slack with Google Sheets logging

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** Daily WooCommerce Sales Snapshot → Slack  
**Purpose:** Every day at a fixed time, fetch WooCommerce orders, keep only paid orders from the last 24 hours, compute key sales metrics (revenue, order count, AOV, top products), then (1) post a summary to Slack and (2) log the metrics into Google Sheets.

### 1.1 Daily Trigger
Runs once per day (09:00) to start the pipeline.

### 1.2 Order Retrieval & Qualification
Fetches all WooCommerce orders, then filters them to “paid” statuses and to only those created in the last 24 hours.

### 1.3 Metrics Computation (Parallel)
From the filtered orders, computes:
- Revenue + Average Order Value (AOV) + order count
- Top 5 products by quantity sold

### 1.4 Consolidation, Notification, Logging
Merges computed metrics into one object, formats it, sends Slack message, and appends/updates a row in Google Sheets for historical tracking.

---

## 2. Block-by-Block Analysis

### Block 1 — Daily Trigger
**Overview:** Starts the workflow automatically every day at 09:00.  
**Nodes involved:** Daily Schedule (plus documentation sticky notes)

#### Node: Daily Schedule
- **Type / role:** `Schedule Trigger` — entry point, time-based execution.
- **Key configuration:** Runs daily at **hour = 9** (server timezone unless n8n instance timezone is configured).
- **Outputs:** Fans out to:
  - **Fetch WooCommerce Orders** (main processing branch)
  - **Merge Metrics** (input #3, “baseline” empty input; see edge cases)
- **Edge cases / failures:**
  - **Timezone mismatch:** If n8n instance timezone differs from business timezone, the “daily” snapshot may not align with expectations.
  - **DST transitions:** May cause drift if not handled at instance level.

**Sticky note coverage (context):**
- “Daily Workflow Trigger” and “How This Workflow Works” describe the daily execution and overall steps.

---

### Block 2 — Fetch & Filter Orders
**Overview:** Pulls orders from WooCommerce, keeps only paid statuses (processing/completed), then limits to last 24 hours using the order creation timestamp.  
**Nodes involved:** Fetch WooCommerce Orders → Filter Paid Orders → Last 24h Orders

#### Node: Fetch WooCommerce Orders
- **Type / role:** `WooCommerce` — retrieves order data from WooCommerce REST API.
- **Configuration choices:**
  - **Resource:** Order
  - **Operation:** Get All
  - **Return all:** Enabled (returns all orders available under API/pagination constraints).
- **Credentials:** `wooCommerceApi` (must be configured: store URL + consumer key/secret).
- **Input:** From **Daily Schedule**.
- **Output:** To **Filter Paid Orders**.
- **Edge cases / failures:**
  - **Large stores:** “Return all” can be slow, memory-heavy, and hit API rate limits/timeouts. Prefer server-side filtering by date/status if available.
  - **Auth failures:** invalid consumer key/secret, missing permissions.
  - **Pagination limits:** Some WooCommerce setups impose stricter limits; n8n handles pagination but still may be heavy.

#### Node: Filter Paid Orders
- **Type / role:** `Filter` — keeps only orders considered “paid”.
- **Configuration choices:**
  - Condition group uses **OR**:
    - `$json.status == "processing"`
    - `$json.status == "completed"`
  - Strict type validation enabled by node defaults.
- **Input:** From **Fetch WooCommerce Orders** (each item is an order).
- **Output:** To **Last 24h Orders**.
- **Edge cases / failures:**
  - **Status taxonomy:** Stores may use custom statuses (e.g., `on-hold`, `paid`, `wc-processing`). You may need to expand conditions.
  - **Refunded orders:** This filter does not exclude refunded/partially refunded orders unless their status changes.

#### Node: Last 24h Orders
- **Type / role:** `Code` — filters items to those created within the last 24 hours.
- **Key logic (interpreted):**
  - Computes `cutoff = now - 24h`
  - Keeps items where `new Date(order.date_created) >= cutoff`
- **Input:** Paid orders from **Filter Paid Orders**.
- **Outputs:** Sends filtered orders to both:
  - **Calculate Revenue + AOV**
  - **Calculate Top Products**
- **Edge cases / failures:**
  - **Timezone interpretation:** `date_created` may be local store time vs UTC depending on WooCommerce API fields (`date_created` vs `date_created_gmt`). Using `date_created` can create off-by-hours errors.
  - **Date parsing:** If `date_created` is missing or in an unexpected format, `new Date(...)` can yield `Invalid Date` and filter incorrectly.
  - **“Last 24 hours” vs “previous calendar day”:** This implements rolling 24h, not “yesterday”.

**Sticky note coverage (context):**
- “Fetch & Filter Orders” describes fetching, paid filtering, and last-24h selection.

---

### Block 3 — Sales Metrics Calculation & Merge
**Overview:** Calculates sales KPIs in parallel branches, then merges results into a single dataset for downstream Slack + Sheets.  
**Nodes involved:** Calculate Revenue + AOV, Calculate Top Products, Merge Metrics, Finalize Data

#### Node: Calculate Revenue + AOV
- **Type / role:** `Code` — aggregates revenue, counts orders, derives AOV, captures currency.
- **Key logic (interpreted):**
  - `revenue += parseFloat(order.total || 0)`
  - `count = items.length`
  - `currency = first non-null order.currency`
  - `aov = revenue / count` (0 if no orders)
  - Returns a single item: `{ totalRevenue, orderCount, aov, currency }` formatted with `toFixed(2)` for numeric strings.
- **Input:** Filtered orders from **Last 24h Orders**.
- **Output:** To **Merge Metrics** as input #2 (index 1).
- **Edge cases / failures:**
  - **Revenue correctness:** Uses `order.total` (string). Doesn’t account for refunds, fees, shipping, taxes breakdown; it uses whatever WooCommerce total represents.
  - **Multi-currency stores:** Assumes one currency; if orders have different currencies within 24h, the first currency is used and totals are mixed incorrectly.
  - **Floating precision:** `toFixed(2)` produces strings; downstream Sheets schema uses strings so it’s acceptable, but numeric analytics may be harder.

#### Node: Calculate Top Products
- **Type / role:** `Code` — aggregates product quantities across orders and returns top 5.
- **Key logic (interpreted):**
  - Iterates `order.line_items[]`
  - Sums quantities per `line.name`
  - Sorts desc by quantity, slices top 5
  - Returns one item: `{ topProducts: [[name, qty], ...] }`
- **Input:** Filtered orders from **Last 24h Orders**.
- **Output:** To **Merge Metrics** as input #1 (index 0).
- **Edge cases / failures:**
  - **Product identity:** Uses `line.name` as key; variations or renamed products will fragment counts. Prefer `product_id` / `variation_id` if you need stable identifiers.
  - **Missing line_items:** Safely handled by `|| []`.
  - **Ties / ordering:** Deterministic by sort order; ties may appear arbitrary.

#### Node: Merge Metrics
- **Type / role:** `Merge` — combines metrics into a single multi-item stream.
- **Configuration choices:**
  - **Mode:** Combine
  - **Combine by:** Position
  - **Number of inputs:** 3
- **Inputs:**
  1. From **Calculate Top Products** (index 0)
  2. From **Calculate Revenue + AOV** (index 1)
  3. Directly from **Daily Schedule** (index 2)
- **Output:** To **Finalize Data**.
- **Important behavior note:**
  - The third input from **Daily Schedule** provides an item with the trigger payload (often empty JSON). With “combine by position”, the merge expects aligned item positions across inputs. Since both code nodes return **one** item, the schedule trigger should also contribute **one** item, which generally holds.
- **Edge cases / failures:**
  - **Misaligned item counts:** If any branch returns 0 items, merge may output 0 items. For example, if a code node returns no items due to an error or empty input handling change, the workflow stops before Slack/Sheets.
  - **Unnecessary third input:** It is not used later but can influence merge behavior.

#### Node: Finalize Data
- **Type / role:** `Code` — normalizes merged output into a single consolidated object.
- **Key logic (interpreted):**
  - Iterates over merged items and merges JSON objects via `{ ...result, ...i.json }`
  - Returns one item: `{ topProducts, totalRevenue, orderCount, aov, currency, ... }`
- **Input:** From **Merge Metrics**.
- **Outputs:** Parallel to:
  - **Append or update row in sheet**
  - **Send Slack Summary**
- **Edge cases / failures:**
  - **Key collisions:** Later items override earlier keys if they share names. Currently unlikely unless trigger adds same keys.
  - **Data types:** `topProducts` is an array; Sheets node maps it into a string column—may serialize as `[object Object]` depending on n8n’s conversion rules unless explicitly stringified.

**Sticky note coverage (context):**
- “Sales Metrics Calculation & Merge” describes metrics computed and merged.

---

### Block 4 — Sales Summary & Logging
**Overview:** Sends the computed snapshot to Slack and logs the same metrics to Google Sheets (append or update by timestamp).  
**Nodes involved:** Send Slack Summary, Append or update row in sheet

#### Node: Send Slack Summary
- **Type / role:** `Slack` — posts a formatted message to a Slack channel.
- **Configuration choices:**
  - **Operation:** Send message (via `text` field).
  - **Channel:** Selected by ID `C0A3WQEKQ58` (cached name: `n8n-demo`).
  - **Message template (key expressions):**
    - `Orders: {{$json.orderCount}}`
    - `Revenue: {{$json.currency}} {{$json.totalRevenue}}`
    - `AOV: {{$json.currency}} {{$json.aov}}`
    - Top products list built by JS expression:
      - `{{$json.topProducts.map(p => `• ${p[0]} (${p[1]})`).join('\n')}}`
- **Input:** From **Finalize Data**.
- **Output:** None (terminal).
- **Credentials:** `slackApi` (OAuth token / bot token with chat:write to target channel).
- **Edge cases / failures:**
  - **Channel access:** Bot must be in the channel; otherwise “not_in_channel”.
  - **topProducts missing:** If `topProducts` is undefined, `.map` will throw. (This can happen if merge produced an item without that key.)
  - **Message formatting:** Slack markdown is used; long product lists or special characters may need escaping.

#### Node: Append or update row in sheet
- **Type / role:** `Google Sheets` — logs daily metrics in a spreadsheet.
- **Configuration choices:**
  - **Operation:** Append or Update
  - **Document:** “Daily WooCommerce Sales Snapshot → Slack” (by ID)
  - **Sheet:** `Sheet1` (gid=0)
  - **Matching column:** `timestamp`
  - **Columns mapped:**
    - `aov = {{$json.aov}}`
    - `currency = {{$json.currency}}`
    - `timezone = {{$json.Timezone}}`  *(note the capital T)*
    - `timestamp = {{$json.timestamp}}`
    - `orderCount = {{$json.orderCount}}`
    - `topProducts = {{$json.topProducts}}`
    - `totalRevenue = {{$json.totalRevenue}}`
- **Input:** From **Finalize Data**.
- **Output:** None (terminal).
- **Credentials:** `googleSheetsOAuth2Api`.
- **Edge cases / failures:**
  - **timestamp not set:** This workflow never creates `$json.timestamp`, so matching will be empty/undefined; “appendOrUpdate” may fail, update the wrong row, or append duplicates depending on node behavior.
  - **Timezone not set:** Uses `$json.Timezone` (capitalized), but no node sets it. Likely results in blank.
  - **topProducts serialization:** Writing an array directly often results in an unreadable string. Usually you want `JSON.stringify($json.topProducts)` or a human-readable join.
  - **Sheet schema mismatch:** Column headers must exist and match exactly.

**Sticky note coverage (context):**
- “Sales Summary & Logging” explains Slack visibility + Sheets historical log.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Daily Schedule | Schedule Trigger | Daily entry point at 09:00 | — | Fetch WooCommerce Orders; Merge Metrics | ## Daily Workflow Trigger  Starts the workflow automatically each day using a Schedule Trigger. It ensures daily sales data is collected, processed and reported consistently without any manual intervention. |
| Fetch WooCommerce Orders | WooCommerce | Retrieve all orders from WooCommerce | Daily Schedule | Filter Paid Orders | ## Fetch & Filter Orders This section fetches orders from WooCommerce, keeps only paid orders and filters them to include orders from the last 24 hours. This helps ensure that only valid and recent sales data is used for revenue calculation. |
| Filter Paid Orders | Filter | Keep only paid statuses (processing/completed) | Fetch WooCommerce Orders | Last 24h Orders | ## Fetch & Filter Orders This section fetches orders from WooCommerce, keeps only paid orders and filters them to include orders from the last 24 hours. This helps ensure that only valid and recent sales data is used for revenue calculation. |
| Last 24h Orders | Code | Filter orders to last 24 hours | Filter Paid Orders | Calculate Revenue + AOV; Calculate Top Products | ## Fetch & Filter Orders This section fetches orders from WooCommerce, keeps only paid orders and filters them to include orders from the last 24 hours. This helps ensure that only valid and recent sales data is used for revenue calculation. |
| Calculate Revenue + AOV | Code | Compute revenue, order count, AOV, currency | Last 24h Orders | Merge Metrics | ## Sales Metrics Calculation & Merge  Here the workflow calculates key metrics using separate Code nodes: • Total Revenue • Order Count • Average Order Value (AOV) • Top Selling Products  All results are merged and normalized into a single object for easy reuse in alerts, reports or future extensions. |
| Calculate Top Products | Code | Compute top 5 products by quantity | Last 24h Orders | Merge Metrics | ## Sales Metrics Calculation & Merge  Here the workflow calculates key metrics using separate Code nodes: • Total Revenue • Order Count • Average Order Value (AOV) • Top Selling Products  All results are merged and normalized into a single object for easy reuse in alerts, reports or future extensions. |
| Merge Metrics | Merge | Combine outputs of metric branches | Calculate Top Products; Calculate Revenue + AOV; Daily Schedule | Finalize Data | ## Sales Metrics Calculation & Merge  Here the workflow calculates key metrics using separate Code nodes: • Total Revenue • Order Count • Average Order Value (AOV) • Top Selling Products  All results are merged and normalized into a single object for easy reuse in alerts, reports or future extensions. |
| Finalize Data | Code | Consolidate merged items into one JSON | Merge Metrics | Append or update row in sheet; Send Slack Summary | ## Sales Summary & Logging  This section sends a clean, human-readable sales summary to Slack and records the same data in Google Sheets.  Includes: • Total revenue • Number of orders • Average order value (AOV) • Top selling products  Slack provides instant visibility into daily performance, while Google Sheets maintains a historical log for tracking trends and reporting—without logging into dashboards. |
| Send Slack Summary | Slack | Post formatted daily snapshot to Slack channel | Finalize Data | — | ## Sales Summary & Logging  This section sends a clean, human-readable sales summary to Slack and records the same data in Google Sheets.  Includes: • Total revenue • Number of orders • Average order value (AOV) • Top selling products  Slack provides instant visibility into daily performance, while Google Sheets maintains a historical log for tracking trends and reporting—without logging into dashboards. |
| Append or update row in sheet | Google Sheets | Log metrics into Google Sheets | Finalize Data | — | ## Sales Summary & Logging  This section sends a clean, human-readable sales summary to Slack and records the same data in Google Sheets.  Includes: • Total revenue • Number of orders • Average order value (AOV) • Top selling products  Slack provides instant visibility into daily performance, while Google Sheets maintains a historical log for tracking trends and reporting—without logging into dashboards. |
| Sticky Note | Sticky Note | Documentation block | — | — | ## Daily Workflow Trigger  Starts the workflow automatically each day using a Schedule Trigger. It ensures daily sales data is collected, processed and reported consistently without any manual intervention. |
| Sticky Note1 | Sticky Note | Documentation block | — | — | ## Fetch & Filter Orders This section fetches orders from WooCommerce, keeps only paid orders and filters them to include orders from the last 24 hours. This helps ensure that only valid and recent sales data is used for revenue calculation. |
| Sticky Note2 | Sticky Note | Documentation block | — | — | ## Sales Metrics Calculation & Merge  Here the workflow calculates key metrics using separate Code nodes: • Total Revenue • Order Count • Average Order Value (AOV) • Top Selling Products  All results are merged and normalized into a single object for easy reuse in alerts, reports or future extensions. |
| Sticky Note3 | Sticky Note | Documentation block | — | — | ## Sales Summary & Logging  This section sends a clean, human-readable sales summary to Slack and records the same data in Google Sheets.  Includes: • Total revenue • Number of orders • Average order value (AOV) • Top selling products  Slack provides instant visibility into daily performance, while Google Sheets maintains a historical log for tracking trends and reporting—without logging into dashboards. |
| Sticky Note4 | Sticky Note | Documentation block | — | — | ## How This Workflow Works  1. The workflow runs automatically once every day using a Schedule Trigger (Cron).  2. It fetches recent orders from WooCommerce as the source sales data.  3. Only paid orders (Completed / Processing) are considered to ensure accurate reporting.  4. Orders created within the last 24 hours are filtered using a Code node.  5. Separate Code nodes calculate key sales metrics such as:    • Total Revenue    • Order Count    • Average Order Value (AOV)    • Top Selling Products  6. All calculated metrics are merged and formatted into a single consolidated object.  7. A clean and readable daily sales summary is sent to a Slack channel,    giving the team instant visibility into sales performance.  8. The same sales data is also appended as a new row in Google Sheets,    creating a daily log for tracking trends and historical analysis.  ---  ## How to Set Up This Workflow  Add a Schedule Trigger to run the workflow daily.  Add a WooCommerce node to fetch all orders.  Filter orders by Completed / Processing status.  Use a Code node to keep only orders from the last 24 hours.  Add separate Code nodes to calculate revenue, order count, AOV and top products.  Use a Merge node to combine all calculated metrics.  Add a final Code node to normalize the merged data into one object.  Connect a Slack node to send the daily sales summary.  Connect a Google Sheets node to append the daily metrics as a new row.  Test the workflow and activate it. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** named: “Daily WooCommerce Sales Snapshot → Slack”.

2. **Add node: Schedule Trigger**
   - Node type: **Schedule Trigger**
   - Set rule to run **daily at 09:00** (or your desired hour).
   - Keep note of instance timezone (Settings → Timezone).

3. **Add node: WooCommerce**
   - Node type: **WooCommerce**
   - Resource: **Order**
   - Operation: **Get All**
   - Return All: **true**
   - Credentials:
     - Create/assign **WooCommerce API** credentials (Store URL + Consumer Key + Consumer Secret).
   - Connect: **Schedule Trigger → WooCommerce**

4. **Add node: Filter**
   - Node type: **Filter**
   - Condition group: **OR**
   - Conditions:
     - `{{$json.status}}` **equals** `processing`
     - `{{$json.status}}` **equals** `completed`
   - Connect: **WooCommerce → Filter**

5. **Add node: Code** (“Last 24h Orders”)
   - Node type: **Code**
   - Paste logic to filter by date:
     - Cutoff = now - 24h
     - Keep items where `date_created >= cutoff`
   - Connect: **Filter → Code**

6. **Add node: Code** (“Calculate Revenue + AOV”)
   - Node type: **Code**
   - Implement aggregation:
     - Sum `order.total`
     - Count items
     - Currency from first order
     - AOV = revenue / count
     - Return single item with `{ totalRevenue, orderCount, aov, currency }`
   - Connect: **Last 24h Orders → Calculate Revenue + AOV**

7. **Add node: Code** (“Calculate Top Products”)
   - Node type: **Code**
   - Implement aggregation:
     - For each order, iterate `line_items`
     - Sum `quantity` by product name
     - Sort and keep top 5
     - Return `{ topProducts }` as a single item
   - Connect: **Last 24h Orders → Calculate Top Products**

8. **Add node: Merge** (“Merge Metrics”)
   - Node type: **Merge**
   - Mode: **Combine**
   - Combine by: **Position**
   - Number of inputs: **3**
   - Connect:
     - **Calculate Top Products → Merge (input 1)**
     - **Calculate Revenue + AOV → Merge (input 2)**
     - **Schedule Trigger → Merge (input 3)** (optional; only needed if you want the trigger payload merged—otherwise omit and set inputs=2)

9. **Add node: Code** (“Finalize Data”)
   - Node type: **Code**
   - Merge all incoming items’ JSON into a single object and return one item.
   - Connect: **Merge Metrics → Finalize Data**

10. **Add node: Slack** (“Send Slack Summary”)
   - Node type: **Slack**
   - Action: send message to channel
   - Select the target channel.
   - Message text uses expressions for orderCount, totalRevenue, aov, currency, and formats `topProducts`.
   - Credentials:
     - Create/assign **Slack API** credentials (bot token/OAuth) with permission to post in the channel.
   - Connect: **Finalize Data → Slack**

11. **Add node: Google Sheets** (“Append or update row in sheet”)
   - Node type: **Google Sheets**
   - Operation: **Append or Update**
   - Document: choose/create your spreadsheet
   - Sheet: select `Sheet1` (or your target)
   - Matching column: `timestamp`
   - Map columns for `totalRevenue`, `orderCount`, `aov`, `currency`, `topProducts`, `timestamp`, `timezone`
   - Credentials:
     - Create/assign **Google Sheets OAuth2** credentials.
   - Connect: **Finalize Data → Google Sheets**

12. **Create the sheet headers** in Google Sheets (first row) exactly matching the mapped column names:
   - `topProducts, totalRevenue, orderCount, aov, currency, timestamp, timezone`

13. **Test run** the workflow manually:
   - Ensure WooCommerce returns orders.
   - Verify Slack message renders and top products list formats.
   - Verify Sheets row is appended/updated as expected.

14. **Activate** the workflow.

**Important fix required to match current mappings:** Add a timestamp/timezone generator before logging (e.g., another Code node after “Finalize Data” that sets `timestamp` and `timezone`) or adjust the Google Sheets node to not use `timestamp` as the matching column.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “How This Workflow Works” (full 1–8 step description + setup bullets) | Included as a workflow sticky note; documents intended behavior and setup steps. |
| Current implementation does **not** set `timestamp` or `Timezone` fields, but Google Sheets mapping expects them (`$json.timestamp`, `$json.Timezone`). | Potential mismatch requiring adjustment in code or Sheets mapping. |
| “Last 24 hours” filter is rolling 24h and uses `date_created` parsing; consider `date_created_gmt` for consistency. | Data correctness across timezones and DST. |
| Merge node has 3 inputs including Schedule Trigger; this is not strictly necessary and can affect merge output if any branch returns 0 items. | Stability/maintainability consideration. |