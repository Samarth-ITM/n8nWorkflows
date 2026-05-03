Send weekly WooCommerce sales KPIs to Slack and Google Sheets

https://n8nworkflows.xyz/workflows/send-weekly-woocommerce-sales-kpis-to-slack-and-google-sheets-13281


# Send weekly WooCommerce sales KPIs to Slack and Google Sheets

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** This workflow runs weekly to generate WooCommerce sales performance KPIs for the *previous full week (Monday–Sunday)*, stores them in Google Sheets for historical tracking, and posts a formatted summary to a Slack channel.

**Typical use cases**
- Weekly leadership/store performance reporting (orders, revenue, AOV, refunds, top products)
- Lightweight KPI logging into a spreadsheet for trend analysis
- Automated Slack notifications without manual exports

### 1.1 Trigger & Time Window Computation
Runs every week at a fixed day/time and computes the last week’s start/end ISO timestamps.

### 1.2 Store Configuration (Single Source of Truth)
Defines the WooCommerce domain once and fans out to all WooCommerce API calls.

### 1.3 WooCommerce Data Collection
Fetches weekly orders and weekly refunds (and re-fetches orders again for top products calculation).

### 1.4 KPI Computation
Calculates: total orders, total revenue, average order value, refund count, refund amount, and top 5 products by revenue.

### 1.5 Consolidation & Report Shaping
Merges all KPI outputs into one item and prepares a clean, consistent set of fields.

### 1.6 Persistence & Notification
Appends the KPI row to Google Sheets and sends the summary to Slack.

---

## 2. Block-by-Block Analysis

### Block 1 — Trigger & Store Setup
**Overview:** Starts weekly, computes last week’s Monday–Sunday window, and sets a reusable WooCommerce domain used by all HTTP calls.

**Nodes involved**
- Weekly Sales KPI Trigger
- Calculate Last Week Date Range
- Configure WooCommerce Store

#### Node: **Weekly Sales KPI Trigger**
- **Type / role:** Schedule Trigger — workflow entry point.
- **Configuration:** Weekly interval; `triggerAtDay: [1]`, `triggerAtHour: 10` (runs Mondays at 10:00 in the n8n instance timezone).
- **Output:** Emits a timestamp (`$json.timestamp`) used downstream.
- **Connections:** → Calculate Last Week Date Range
- **Edge cases / failures:**
  - Timezone mismatch (instance timezone vs business timezone) can shift the “previous week” window.
  - Day-of-week interpretation: `triggerAtDay: 1` corresponds to Monday in n8n schedule semantics.

#### Node: **Calculate Last Week Date Range**
- **Type / role:** Code — computes last week start/end in ISO format.
- **Key logic:**
  - Reads `runDate = new Date($json.timestamp)`.
  - Finds the most recent Monday relative to run date, then subtracts 7 days to get **last Monday 00:00:00.000**.
  - Computes **last Sunday 23:59:59.999**.
  - Outputs:
    - `lastWeekStart` (ISO string)
    - `lastWeekEnd` (ISO string)
- **Connections:**
  - → Configure WooCommerce Store
  - → Merge Weekly KPI Results (input index 3)
- **Edge cases / failures:**
  - ISO timestamps are UTC; WooCommerce “after/before” filtering can behave unexpectedly if your store timezone differs.
  - If the trigger runs very close to midnight and timezones differ, “last week” boundaries may not align with local expectations.

#### Node: **Configure WooCommerce Store**
- **Type / role:** Set — central configuration holder.
- **Configuration choices:**
  - Sets `wc_domain` (string) **currently empty**; must be filled (e.g., `https://example.com`).
- **Connections (fan-out):**
  - → Get Weekly Orders (Sales Data)
  - → Get Weekly Refunds
  - → Get Weekly Top Products
- **Edge cases / failures:**
  - Empty domain will generate invalid URLs and hard-fail HTTP nodes.
  - Domain must include protocol (`https://`) to avoid malformed requests.

**Sticky note covering this block**
- “## Workflow Trigger & Store Setup …” (explains schedule, date range, and centralized domain configuration).

---

### Block 2 — Weekly Data Collection & KPI Calculation
**Overview:** Pulls order and refund data from WooCommerce for the computed date range, then computes KPIs from the returned datasets.

**Nodes involved**
- Get Weekly Orders (Sales Data)
- Calculate Order & Revenue KPIs
- Get Weekly Refunds
- Calculate Refund KPIs
- Get Weekly Top Products
- Calculate Top  Products by Revenue

#### Node: **Get Weekly Orders (Sales Data)**
- **Type / role:** HTTP Request — fetches orders for the weekly window.
- **Request:**
  - **URL:** `https://{{$json.wc_domain}}/wp-json/wc/v3/orders`
  - **Auth:** HTTP Basic Auth (via generic credential type)
  - **Query params:**
    - `after` = `$('Calculate Last Week Date Range').item.json.lastWeekStart`
    - `before` = `$('Calculate Last Week Date Range').item.json.lastWeekEnd`
    - `status` = `completed,processing`
- **Connections:** → Calculate Order & Revenue KPIs
- **Version notes:** HTTP Request node v4.3.
- **Edge cases / failures:**
  - WooCommerce REST API v3 requires valid consumer key/secret; 401 if wrong.
  - Pagination: WooCommerce may return only the first page (default per_page often 10). This workflow does **not** set `per_page` nor paginate, so KPIs can be incorrect for stores with >1 page of weekly orders.
  - `status=completed,processing` as a single value relies on WooCommerce interpretation; many setups expect repeated params or a list format. Validate with your store.

#### Node: **Calculate Order & Revenue KPIs**
- **Type / role:** Code — aggregates order count and revenue.
- **Logic:**
  - Iterates `items` (each item is an order).
  - Includes only orders where:
    - `status` is `completed` or `processing`
    - `line_items` exists and non-empty
    - `total > 0`
  - Outputs a single item with:
    - `total_orders`
    - `total_revenue`
    - `average_order_value` (`total_revenue / total_orders`, else 0)
- **Connections:** → Merge Weekly KPI Results (input index 0)
- **Edge cases / failures:**
  - If the HTTP node returns one item with an array (depending on settings), this code expects multiple items; ensure HTTP Request returns one item per order (n8n often does when “Split into Items” is enabled, but it is not explicit here).
  - Currency formatting is not handled; revenue is numeric sum of `order.total` strings.

#### Node: **Get Weekly Refunds**
- **Type / role:** HTTP Request — fetches refunds during the date window.
- **Request:**
  - **URL:** `https://{{ $('Configure WooCommerce Store').item.json.wc_domain }}/wp-json/wc/v3/refunds`
  - **Auth:** HTTP Basic Auth
  - **Query params:** `after`, `before` from “Calculate Last Week Date Range”
- **Special setting:** `alwaysOutputData: true` (prevents workflow from stopping when no refunds are found; node still outputs data).
- **Connections:** → Calculate Refund KPIs
- **Edge cases / failures:**
  - Pagination risk (same as orders).
  - Refund “after/before” semantics may differ by WooCommerce version (created date vs processed date).
  - If API returns empty, downstream code should still handle it (it does).

#### Node: **Calculate Refund KPIs**
- **Type / role:** Code — computes refund count and refund amount.
- **Logic:**
  - For each refund item:
    - increments `refundCount`
    - adds `amount` if positive
    - if `line_items` exist, sums `Math.abs(li.total)` into refundAmount (treating negative totals as refund magnitude)
  - Outputs:
    - `refund_count`
    - `refund_amount`
- **Connections:** → Merge Weekly KPI Results (input index 1)
- **Edge cases / failures:**
  - Potential double counting: adding both `amount` and `abs(line_item.total)` can overstate refunds if both represent the same refund value.
  - Refund objects in WooCommerce may not include `line_items` depending on configuration/plugins.

#### Node: **Get Weekly Top Products**
- **Type / role:** HTTP Request — fetches weekly orders again for top product computation.
- **Request:** Same endpoint/params as “Get Weekly Orders (Sales Data)”.
- **Connections:** → Calculate Top  Products by Revenue
- **Edge cases / failures:**
  - Redundant API call: increases load and doubles pagination risk.
  - Same pagination and `status` formatting considerations as above.

#### Node: **Calculate Top  Products by Revenue**
- **Type / role:** Code — builds top 5 products by revenue.
- **Logic:**
  - Iterates orders and their `line_items`
  - Aggregates per `product_id`:
    - `product_name`
    - `total_quantity`
    - `total_revenue` (sum of line item `total`)
  - Sorts by `total_revenue` descending, takes top 5.
  - Formats as text: `1. Name (₹revenue), ...`
  - Outputs single field:
    - `top_products` (string)
- **Connections:** → Merge Weekly KPI Results (input index 2)
- **Edge cases / failures:**
  - Currency symbol hard-coded as `₹` regardless of store currency.
  - Variation products may have different IDs; grouping by `product_id` may not reflect variations cleanly.
  - If line item totals exclude tax/shipping, “top by revenue” may differ from “order revenue”.

**Sticky note covering this block**
- “## Weekly Data Collection & KPI Calculation …”

---

### Block 3 — KPI Consolidation & Report Preparation
**Overview:** Combines all KPI outputs (plus the date range) into a single item and reshapes fields to a consistent schema for Sheets and Slack.

**Nodes involved**
- Merge Weekly KPI Results
- Prepare Final KPI Report Fields

#### Node: **Merge Weekly KPI Results**
- **Type / role:** Merge — combines multiple KPI streams.
- **Configuration:**
  - Mode: `combine`
  - `combineByPosition`
  - `numberInputs: 4`
  - Inputs expected:
    1) Order/Revenue KPIs  
    2) Refund KPIs  
    3) Top products text  
    4) Date range (from Calculate Last Week Date Range)
- **Connections:** → Prepare Final KPI Report Fields
- **Edge cases / failures:**
  - If any upstream branch returns **0 items** (e.g., orders request returns nothing and doesn’t output an item), `combineByPosition` can produce missing/empty results. Refunds branch mitigates this with `alwaysOutputData`, but orders/top-products do not.
  - Timing/branch alignment issues are usually fine in n8n, but missing items will break merged payload completeness.

#### Node: **Prepare Final KPI Report Fields**
- **Type / role:** Set — normalizes final output fields.
- **Configuration:**
  - Creates final fields:
    - `total_orders` (number)
    - `total_revenue` (number)
    - `average_order_value` (number)
    - `refund_count` (number)
    - `refund_amount` (number)
    - `top_products` (string)
    - `lastWeekStart` (string)
    - `lastWeekEnd` (string)
- **Connections:** → Store Weekly KPIs in Google Sheets
- **Edge cases / failures:**
  - Upstream typo propagations: if merged data has wrong key names, expressions resolve to `null`.
  - This node correctly uses `lastWeekStart` (no trailing space), which matters because another node uses `lastWeekStart ` with a trailing space (see below).

**Sticky note covering this block**
- “## KPI Consolidation & Report Preparation …”

---

### Block 4 — Reporting & Notifications
**Overview:** Appends the weekly KPI row to Google Sheets and posts a human-readable report to Slack.

**Nodes involved**
- Store Weekly KPIs in Google Sheets
- Send Weekly KPI Report to Slack

#### Node: **Store Weekly KPIs in Google Sheets**
- **Type / role:** Google Sheets — append KPI row.
- **Operation:** Append to a sheet.
- **Document / Sheet:**
  - Document: “Sales weekely Spike Data” (Google Sheet ID provided)
  - Sheet: “Sheet1” (gid=0)
- **Columns mapping (important details):**
  - Maps typical fields correctly (total_orders, total_revenue, etc.)
  - **Bug/typo present:** uses a column key **`lastWeekStart ` (with trailing space)** in both:
    - `columns.value.lastWeekStart `
    - schema id/displayName `lastWeekStart `
  - Other fields use expected names (`lastWeekEnd`, etc.).
- **Connections:** → Send Weekly KPI Report to Slack
- **Credentials:** Google Sheets OAuth2.
- **Edge cases / failures:**
  - The trailing-space column name can cause:
    - confusion when querying/reporting in Sheets
    - mismatch with Slack formatting node (which uses `lastWeekStart` without space)
  - If the sheet headers don’t match defined schema IDs, append can fail or write into wrong columns.
  - OAuth token expiration/permission issues will fail append.

#### Node: **Send Weekly KPI Report to Slack**
- **Type / role:** Slack — posts message to a channel.
- **Configuration:**
  - Posts to channel “n8n” (`channelId: C09S57E2JQ2`)
  - Message text uses expressions:
    - `Period :  {{ $json["lastWeekStart "] }} to {{ $json.lastWeekEnd }}`
    - totals, refunds, and `top_products`
- **Critical issue:** References **`$json["lastWeekStart "]` (with trailing space)** while the “Prepare Final KPI Report Fields” node creates `lastWeekStart` (no space). Unless an upstream field with the spaced key survives the merge, this will likely render blank.
- **Credentials:** Slack API.
- **Edge cases / failures:**
  - Missing/blank `lastWeekStart ` due to key mismatch.
  - Slack auth/token scopes or channel access limitations.
  - Formatting: revenue is unformatted number; may need rounding.

**Sticky note covering this block**
- “## Reporting & Notifications …”

---

### Block 5 — Documentation / Notes (Sticky Notes)
**Overview:** Non-executing nodes providing in-canvas documentation and setup guidance.

**Nodes involved**
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4

#### Node: **Sticky Note / Sticky Note1 / Sticky Note2 / Sticky Note3 / Sticky Note4**
- **Type / role:** Sticky Note — informational only, no runtime effect.
- **Content highlights:**
  - Describes workflow sections and setup steps (WooCommerce credentials, domain, Google Sheets, Slack, schedule activation).
- **Edge cases:** None (non-executing).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Weekly Sales KPI Trigger | Schedule Trigger | Weekly entry point | — | Calculate Last Week Date Range | ## Workflow Trigger & Store Setup<br>This section starts the workflow on a weekly schedule and calculates the previous week’s date range. It also defines the WooCommerce store domain in one place, ensuring all API requests use consistent dates and store configuration for accurate weekly reporting. |
| Calculate Last Week Date Range | Code | Compute last Monday–Sunday ISO range | Weekly Sales KPI Trigger | Configure WooCommerce Store; Merge Weekly KPI Results | ## Workflow Trigger & Store Setup<br>This section starts the workflow on a weekly schedule and calculates the previous week’s date range. It also defines the WooCommerce store domain in one place, ensuring all API requests use consistent dates and store configuration for accurate weekly reporting. |
| Configure WooCommerce Store | Set | Store wc_domain config | Calculate Last Week Date Range | Get Weekly Orders (Sales Data); Get Weekly Refunds; Get Weekly Top Products | ## Workflow Trigger & Store Setup<br>This section starts the workflow on a weekly schedule and calculates the previous week’s date range. It also defines the WooCommerce store domain in one place, ensuring all API requests use consistent dates and store configuration for accurate weekly reporting. |
| Get Weekly Orders (Sales Data) | HTTP Request | Fetch weekly orders (completed/processing) | Configure WooCommerce Store | Calculate Order & Revenue KPIs | ## Weekly Data Collection & KPI Calculation<br>This section fetches weekly sales orders and refunds from WooCommerce, then calculates key performance metrics such as total orders, revenue, average order value, refunds and top products by revenue. These KPIs form the core insights of the weekly store performance report. |
| Calculate Order & Revenue KPIs | Code | Aggregate order count, revenue, AOV | Get Weekly Orders (Sales Data) | Merge Weekly KPI Results | ## Weekly Data Collection & KPI Calculation<br>This section fetches weekly sales orders and refunds from WooCommerce, then calculates key performance metrics such as total orders, revenue, average order value, refunds and top products by revenue. These KPIs form the core insights of the weekly store performance report. |
| Get Weekly Refunds | HTTP Request | Fetch weekly refunds | Configure WooCommerce Store | Calculate Refund KPIs | ## Weekly Data Collection & KPI Calculation<br>This section fetches weekly sales orders and refunds from WooCommerce, then calculates key performance metrics such as total orders, revenue, average order value, refunds and top products by revenue. These KPIs form the core insights of the weekly store performance report. |
| Calculate Refund KPIs | Code | Aggregate refund count and amount | Get Weekly Refunds | Merge Weekly KPI Results | ## Weekly Data Collection & KPI Calculation<br>This section fetches weekly sales orders and refunds from WooCommerce, then calculates key performance metrics such as total orders, revenue, average order value, refunds and top products by revenue. These KPIs form the core insights of the weekly store performance report. |
| Get Weekly Top Products | HTTP Request | Fetch weekly orders for top products calc | Configure WooCommerce Store | Calculate Top  Products by Revenue | ## Weekly Data Collection & KPI Calculation<brThis section fetches weekly sales orders and refunds from WooCommerce, then calculates key performance metrics such as total orders, revenue, average order value, refunds and top products by revenue. These KPIs form the core insights of the weekly store performance report. |
| Calculate Top  Products by Revenue | Code | Compute top 5 products by revenue (text) | Get Weekly Top Products | Merge Weekly KPI Results | ## Weekly Data Collection & KPI Calculation<br>This section fetches weekly sales orders and refunds from WooCommerce, then calculates key performance metrics such as total orders, revenue, average order value, refunds and top products by revenue. These KPIs form the core insights of the weekly store performance report. |
| Merge Weekly KPI Results | Merge | Combine 4 KPI streams into one item | Calculate Order & Revenue KPIs; Calculate Refund KPIs; Calculate Top  Products by Revenue; Calculate Last Week Date Range | Prepare Final KPI Report Fields | ## KPI Consolidation & Report Preparation<br>This section combines all calculated KPIs into a single dataset and formats the data into a clean, report-ready structure. Only essential fields are retained, ensuring the final output is consistent and suitable for storage and stakeholder communication. |
| Prepare Final KPI Report Fields | Set | Normalize final schema for reporting | Merge Weekly KPI Results | Store Weekly KPIs in Google Sheets | ## KPI Consolidation & Report Preparation<br>This section combines all calculated KPIs into a single dataset and formats the data into a clean, report-ready structure. Only essential fields are retained, ensuring the final output is consistent and suitable for storage and stakeholder communication. |
| Store Weekly KPIs in Google Sheets | Google Sheets | Append KPI row for history | Prepare Final KPI Report Fields | Send Weekly KPI Report to Slack | ## Reporting & Notifications<br>This section saves the finalized weekly KPI data to Google Sheets for historical tracking and analysis, then sends a summarized performance report to Slack. This ensures stakeholders receive timely updates while maintaining a long-term record of weekly performance. |
| Send Weekly KPI Report to Slack | Slack | Post weekly KPI summary to Slack channel | Store Weekly KPIs in Google Sheets | — | ## Reporting & Notifications<br>This section saves the finalized weekly KPI data to Google Sheets for historical tracking and analysis, then sends a summarized performance report to Slack. This ensures stakeholders receive timely updates while maintaining a long-term record of weekly performance. |
| Sticky Note | Sticky Note | Documentation | — | — | ## Workflow Trigger & Store Setup<br>This section starts the workflow on a weekly schedule and calculates the previous week’s date range. It also defines the WooCommerce store domain in one place, ensuring all API requests use consistent dates and store configuration for accurate weekly reporting. |
| Sticky Note1 | Sticky Note | Documentation | — | — | ## Weekly Data Collection & KPI Calculation<br>This section fetches weekly sales orders and refunds from WooCommerce, then calculates key performance metrics such as total orders, revenue, average order value, refunds and top products by revenue. These KPIs form the core insights of the weekly store performance report. |
| Sticky Note2 | Sticky Note | Documentation | — | — | ## KPI Consolidation & Report Preparation<br>This section combines all calculated KPIs into a single dataset and formats the data into a clean, report-ready structure. Only essential fields are retained, ensuring the final output is consistent and suitable for storage and stakeholder communication. |
| Sticky Note3 | Sticky Note | Documentation | — | — | ## Reporting & Notifications<br>This section saves the finalized weekly KPI data to Google Sheets for historical tracking and analysis, then sends a summarized performance report to Slack. This ensures stakeholders receive timely updates while maintaining a long-term record of weekly performance. |
| Sticky Note4 | Sticky Note | Documentation / setup instructions | — | — | ## How Workflow works<br>### This workflow automatically creates a weekly performance report for your WooCommerce store. It runs on a fixed schedule, calculates the previous week’s date range and fetches completed and processing orders along with refund data.<br><br>### It computes key metrics such as total orders, total revenue, average order value, refund counts, refund amounts and top-performing products by revenue. All metrics are merged into a single, report-ready dataset.<br><br>### The final report is stored in Google Sheets for historical tracking and sent to a Slack channel, ensuring stakeholders receive timely updates without any manual effort.<br><br>## Workflow Setup Steps<br>### Import the workflow JSON file into your n8n instance and verify that all nodes are connected correctly.<br><br>### Configure WooCommerce credentials using HTTP Basic Authentication with a consumer key and secret that have access to orders and refunds.<br><br>### Update the WooCommerce store domain in the configuration node to match your store URL.<br><br>### Connect your Google Sheets account and select the spreadsheet and sheet where weekly KPI data will be stored.<br><br>### Connect your Slack account and choose the channel where the weekly report should be sent.<br><br>### Review the schedule trigger to confirm the correct weekly run time, then activate the workflow.<br><br>### Once activated, the workflow will automatically run every week and send updated KPI reports without manual effort. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name: *WooCommerce Weekly Sales KPI Reporting to Slack & Google Sheets* (or your preferred name)

2) **Add “Weekly Sales KPI Trigger”**
- Node type: **Schedule Trigger**
- Set rule: **Weekly**
  - Day: **Monday**
  - Time: **10:00**
- This is the entry node.

3) **Add “Calculate Last Week Date Range”**
- Node type: **Code**
- Paste logic that outputs:
  - `lastWeekStart` = last Monday 00:00:00.000 (previous week)
  - `lastWeekEnd` = last Sunday 23:59:59.999 (previous week)
- Connect: Trigger → Date Range node.

4) **Add “Configure WooCommerce Store”**
- Node type: **Set**
- Add field:
  - `wc_domain` (string) = `https://your-store.com`
- Connect: Date Range node → Configure node.

5) **Create WooCommerce credentials**
- In n8n Credentials:
  - Credential type: **HTTP Basic Auth**
  - Username = WooCommerce **consumer key**
  - Password = WooCommerce **consumer secret**
- Ensure the WooCommerce REST API keys have read access to **orders** and **refunds**.

6) **Add “Get Weekly Orders (Sales Data)”**
- Node type: **HTTP Request**
- Method: GET
- URL: `https://{{$json.wc_domain}}/wp-json/wc/v3/orders`
- Authentication: **HTTP Basic Auth** (select credential created above)
- Query parameters:
  - `after` = expression referencing `Calculate Last Week Date Range` → `lastWeekStart`
  - `before` = expression referencing `Calculate Last Week Date Range` → `lastWeekEnd`
  - `status` = `completed,processing`
- Connect: Configure node → Get Weekly Orders.

7) **Add “Calculate Order & Revenue KPIs”**
- Node type: **Code**
- Compute:
  - `total_orders`, `total_revenue`, `average_order_value`
- Connect: Get Weekly Orders → Calculate Order & Revenue KPIs.

8) **Add “Get Weekly Refunds”**
- Node type: **HTTP Request**
- Method: GET
- URL: `https://{{$json.wc_domain}}/wp-json/wc/v3/refunds`
- Authentication: **HTTP Basic Auth**
- Query parameters: `after`, `before` from date-range node
- Enable: **Always Output Data** (so downstream runs even if empty)
- Connect: Configure node → Get Weekly Refunds.

9) **Add “Calculate Refund KPIs”**
- Node type: **Code**
- Compute:
  - `refund_count`, `refund_amount`
- Connect: Get Weekly Refunds → Calculate Refund KPIs.

10) **Add “Get Weekly Top Products”**
- Node type: **HTTP Request**
- Same configuration as “Get Weekly Orders (Sales Data)” (orders endpoint with after/before/status)
- Connect: Configure node → Get Weekly Top Products.

11) **Add “Calculate Top Products by Revenue”**
- Node type: **Code**
- Aggregate `line_items` per `product_id`, sort by revenue desc, take top 5, format to text in `top_products`.
- Connect: Get Weekly Top Products → Calculate Top Products.

12) **Add “Merge Weekly KPI Results”**
- Node type: **Merge**
- Mode: **Combine**
- Combine by: **Position**
- Number of inputs: **4**
- Connect:
  - Calculate Order & Revenue KPIs → Merge (Input 1 / index 0)
  - Calculate Refund KPIs → Merge (Input 2 / index 1)
  - Calculate Top Products → Merge (Input 3 / index 2)
  - Calculate Last Week Date Range → Merge (Input 4 / index 3)

13) **Add “Prepare Final KPI Report Fields”**
- Node type: **Set**
- Create fields (numbers/strings) from merged JSON:
  - `total_orders`, `total_revenue`, `average_order_value`
  - `refund_count`, `refund_amount`
  - `top_products`
  - `lastWeekStart`, `lastWeekEnd`
- Connect: Merge → Prepare Final Fields.

14) **Create Google Sheets credential**
- Credential type: **Google Sheets OAuth2**
- Authenticate account with access to the target spreadsheet.

15) **Add “Store Weekly KPIs in Google Sheets”**
- Node type: **Google Sheets**
- Operation: **Append**
- Select Document and Sheet (e.g., “Sheet1”)
- Define columns to map the prepared fields.
- Connect: Prepare Final Fields → Google Sheets node.
- Important: choose consistent column names (avoid trailing spaces in header IDs).

16) **Create Slack credential**
- Credential type: **Slack API**
- Ensure it can post to the selected channel.

17) **Add “Send Weekly KPI Report to Slack”**
- Node type: **Slack**
- Operation: send message to a channel
- Compose message with expressions referencing the final fields.
- Connect: Google Sheets → Slack.

18) **Activate workflow**
- Confirm schedule and timezone expectations.
- Run once manually to verify:
  - WooCommerce API connectivity
  - correct KPI values
  - Google Sheets row appended correctly
  - Slack message renders correct dates and values

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Sticky notes embedded in the canvas describe each functional section and setup steps (WooCommerce credentials, domain configuration, Google Sheets and Slack connections, schedule review, activation). | From “How Workflow works” and section sticky notes inside the workflow. |
| Known field-name inconsistency: `lastWeekStart` vs `lastWeekStart ` (trailing space) appears in Google Sheets mapping and Slack message expression. Standardize to one key to avoid blank dates. | Affects “Store Weekly KPIs in Google Sheets” and “Send Weekly KPI Report to Slack”. |
| Pagination is not handled for WooCommerce orders/refunds. For stores with more than one page of results, KPIs will be understated unless pagination/per_page is implemented. | Applies to all WooCommerce HTTP Request nodes. |