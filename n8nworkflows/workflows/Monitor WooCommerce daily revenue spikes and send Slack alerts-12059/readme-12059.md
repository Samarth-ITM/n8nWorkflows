Monitor WooCommerce daily revenue spikes and send Slack alerts

https://n8nworkflows.xyz/workflows/monitor-woocommerce-daily-revenue-spikes-and-send-slack-alerts-12059


# Monitor WooCommerce daily revenue spikes and send Slack alerts

## 1. Workflow Overview

**Workflow name:** WooCommerce Daily Revenue Spike Monitor → Slack Alert  
**Purpose:** Runs once per day to evaluate WooCommerce performance over the last 24 hours. It computes (1) paid-order revenue metrics, (2) top products sold, (3) cancellation impact, then posts a Slack message either celebrating a revenue spike (≥ threshold) or reporting progress if below target.

**Primary use cases**
- Daily sales ops visibility in Slack without manually checking WooCommerce dashboards
- Early detection of exceptional revenue spikes
- Monitoring cancellation count and cancelled revenue as a “health signal”

### 1.1 Scheduled Execution
- Triggers automatically daily at a set hour.

### 1.2 Order Retrieval & Paid-Order Filtering (Sales branch)
- Fetches orders from WooCommerce.
- Keeps only *paid/valid* statuses (Processing/Completed).
- Filters to only orders created in the last 24 hours.

### 1.3 24h Sales Analytics
- Calculates total revenue, order count, average order value.
- Calculates top 3 products sold by quantity.

### 1.4 Cancellation Analytics (24h)
- From the same fetched orders list, filters “cancelled”.
- Filters to last 24 hours.
- Calculates cancelled order count and cancelled revenue.

### 1.5 Merge, Format, Threshold Decision & Slack Alerts
- Combines sales metrics + top products + cancellation metrics into one object.
- IF node checks revenue against threshold (100,000).
- Sends either “Sales Spike Alert” or “Target Pending” Slack message.

---

## 2. Block-by-Block Analysis

### Block A — Scheduled Execution
**Overview:** Starts the workflow automatically once per day so reporting is consistent and hands-free.  
**Nodes involved:** `Daily Revenue Spike Monitor`

#### Node: Daily Revenue Spike Monitor
- **Type / role:** Schedule Trigger — entry point to run on a daily schedule.
- **Configuration (interpreted):**
  - Runs daily at **10:00** (server/workflow timezone as configured in n8n instance).
- **Input / Output:**
  - **No input** (trigger).
  - **Output →** `Fetch WooCommerce Orders`
- **Version notes:** Node typeVersion **1.2** (older schedule node versions may have slightly different UI fields).
- **Edge cases / failures:**
  - Timezone mismatch: “10:00” may be unexpected if instance timezone differs from business timezone.
  - If n8n is down at schedule time, execution may be missed depending on n8n setup.

---

### Block B — Fetch & Filter Orders (Sales + Cancellation branches)
**Overview:** Fetches WooCommerce orders, then branches: one branch keeps paid orders for sales metrics, another branch keeps cancelled orders for cancellation analytics.  
**Nodes involved:** `Fetch WooCommerce Orders`, `Filter Paid Orders (Completed / Processing)`, ` Filter Paid Orders (Completed / Processing)`, `Filter Orders from Last 24 Hours`, `Filter Orders from Last 24 Hours2`

#### Node: Fetch WooCommerce Orders
- **Type / role:** WooCommerce node — retrieves order records from WooCommerce REST API.
- **Configuration (interpreted):**
  - **Resource:** Order
  - **Operation:** Get Many (getAll)
  - **Return All:** enabled (pulls all orders available for the query)
  - No explicit date/status filters are applied at API level.
- **Connections:**
  - **Input ←** `Daily Revenue Spike Monitor`
  - **Outputs →**
    - `Filter Paid Orders (Completed / Processing)` (sales branch)
    - ` Filter Paid Orders (Completed / Processing)` (cancellation branch; despite name, it filters cancelled)
- **Credentials:** WooCommerce API credentials required (consumer key/secret or configured credential type in n8n).
- **Edge cases / failures:**
  - Fetching “all orders” can be slow/large, may hit pagination/rate limits, or time out on high-volume stores.
  - Auth failures (bad consumer key/secret, wrong site URL, missing permissions).
  - If WooCommerce returns dates in timezone offset, later JS date parsing must be consistent.

#### Node: Filter Paid Orders (Completed / Processing)
- **Type / role:** Filter node — keeps only paid-like statuses for sales calculations.
- **Configuration (interpreted):**
  - Condition group: **OR**
  - Keep items where `status == "Processing"` OR `status == "completed"`
  - Uses expression: `{{ $json.status }}`
- **Connections:**
  - **Input ←** `Fetch WooCommerce Orders`
  - **Output →** `Filter Orders from Last 24 Hours`
- **Edge cases / failures:**
  - Status case sensitivity: compares against `"Processing"` (capital P) and `"completed"` (lowercase). WooCommerce typically uses lowercase (`processing`, `completed`). If actual status is lowercase, the `"Processing"` check will fail.
  - If store uses custom statuses, orders may be excluded.

#### Node: Filter Orders from Last 24 Hours
- **Type / role:** Code node — filters paid orders to last 24 hours by `date_created`.
- **Configuration (interpreted):**
  - Computes `cutoff = now - 24h`
  - Keeps items where `new Date(order.json.date_created) >= cutoff`
- **Connections:**
  - **Input ←** `Filter Paid Orders (Completed / Processing)`
  - **Outputs →**
    - `Calculate Top Products (24h)`
    - `Calculate Total Revenue (24h)`
- **Edge cases / failures:**
  - Date parsing: relies on `date_created` being a valid ISO date string.
  - If WooCommerce returns `date_created_gmt` and `date_created` differs by timezone, results may not match expectations.
  - If no orders match, downstream code nodes must handle empty input (they mostly do).

#### Node:  Filter Paid Orders (Completed / Processing)  *(note leading space in name)*
- **Type / role:** Filter node — actually filters **cancelled** orders (name is misleading).
- **Configuration (interpreted):**
  - Condition group: **AND** (single condition)
  - Keep items where `status == "cancelled"`
- **Connections:**
  - **Input ←** `Fetch WooCommerce Orders`
  - **Output →** `Filter Orders from Last 24 Hours2`
- **Edge cases / failures:**
  - If WooCommerce status is `cancelled` (lowercase) this matches; if not, it won’t.
  - Node naming may confuse maintainers; consider renaming to “Filter Cancelled Orders”.

#### Node: Filter Orders from Last 24 Hours2
- **Type / role:** Code node — filters cancelled orders to last 24 hours.
- **Configuration:** Same logic as the sales branch 24-hour filter.
- **Connections:**
  - **Input ←** ` Filter Paid Orders (Completed / Processing)` (cancelled filter)
  - **Output →** `Calculate Cancelled Metrics (24h)`
- **Edge cases / failures:** Same date parsing/timezone concerns as the other 24h filter.

---

### Block C — 24-Hour Sales Summary (Revenue + Top Products)
**Overview:** Computes sales KPIs from last-24-hour paid orders, then outputs separate metric objects for later merging.  
**Nodes involved:** `Calculate Total Revenue (24h)`, `Calculate Top Products (24h)`

#### Node: Calculate Total Revenue (24h)
- **Type / role:** Code node — aggregates revenue, order count, average order value.
- **Configuration (interpreted):**
  - Sums `parseFloat(order.total || 0)` across items
  - Captures `currency` from the first order encountered
  - `order_count = items.length`
  - `avg_order_value = totalRevenue / orderCount` (0 if none)
  - Outputs a **single item**:
    - `revenue_last_24h` (2 decimals)
    - `currency`
    - `order_count`
    - `avg_order_value` (2 decimals)
- **Connections:**
  - **Input ←** `Filter Orders from Last 24 Hours`
  - **Output →** `Merge` (as input index 1)
- **Edge cases / failures:**
  - If `total` includes refunds/negative adjustments, revenue may be overstated/understated depending on WooCommerce configuration.
  - Currency may remain `null` if there are zero orders; Slack messages currently hardcode the rupee symbol, not `currency`.

#### Node: Calculate Top Products (24h)
- **Type / role:** Code node — aggregates quantities sold per product name, returns top 3.
- **Configuration (interpreted):**
  - Builds `productMap[name] += quantity` across all `line_items`
  - Sorts descending by quantity
  - Returns top 3 as `top_products: [ [name, qty], ... ]`
  - Outputs a **single item**
- **Connections:**
  - **Input ←** `Filter Orders from Last 24 Hours`
  - **Output →** `Merge` (as input index 0)
- **Edge cases / failures:**
  - Product identity is by **line item name**, not product ID/SKU; name changes or variations may fragment totals.
  - Missing `line_items` are skipped safely.
  - If there are zero orders, `top_products` becomes `[]`; Slack formatting `.map(...).join('\n')` yields an empty string (acceptable).

---

### Block D — Cancellation Monitoring (Last 24 Hours)
**Overview:** Produces cancellation KPIs for last 24 hours: how many orders cancelled and associated cancelled revenue totals.  
**Nodes involved:** `Calculate Cancelled Metrics (24h)`

#### Node: Calculate Cancelled Metrics (24h)
- **Type / role:** Code node — sums totals across cancelled orders.
- **Configuration (interpreted):**
  - `cancelled_orders_24h = items.length`
  - `cancelled_revenue_24h = sum(parseFloat(total || 0))` (2 decimals)
  - Outputs a **single item**
- **Connections:**
  - **Input ←** `Filter Orders from Last 24 Hours2`
  - **Output →** `Merge` (as input index 2)
- **Edge cases / failures:**
  - Cancelled “revenue” is based on `order.total`; depending on store behavior, cancelled orders may include shipping/tax and may not represent actual captured payments.

---

### Block E — Merge, Format, Threshold Check, Slack Notifications
**Overview:** Combines three metric objects into one, checks against revenue threshold, and posts the appropriate Slack message.  
**Nodes involved:** `Merge`, `Format Sales Data`, `Revenue Spike Threshold Check`, `Send Slack Sales Spike Alert`, `Progress Alert (Target Pending)`

#### Node: Merge
- **Type / role:** Merge node — combines outputs from three analytics nodes.
- **Configuration (interpreted):**
  - **Mode:** Combine
  - **Combine by:** Position (index-based)
  - **Number of inputs:** 3
  - Expects:
    - Input 0: top products object
    - Input 1: revenue object
    - Input 2: cancellation object
- **Connections:**
  - **Inputs ←**
    - `Calculate Top Products (24h)` → input 0
    - `Calculate Total Revenue (24h)` → input 1
    - `Calculate Cancelled Metrics (24h)` → input 2
  - **Output →** `Format Sales Data`
- **Edge cases / failures:**
  - If any upstream branch produces **zero items** rather than a single summary item, “combine by position” can produce empty output. (In this workflow, each analytics node always returns exactly one item, so it’s stable.)
  - If a code node errors, merge won’t run.

#### Node: Format Sales Data
- **Type / role:** Code node — flattens merged items into one consolidated JSON object.
- **Configuration (interpreted):**
  - Iterates through incoming items and shallow-merges keys into `finalObject`
  - Returns **one item** with combined fields:
    - `revenue_last_24h`, `currency`, `order_count`, `avg_order_value`, `top_products`, `cancelled_orders_24h`, `cancelled_revenue_24h`
- **Connections:**
  - **Input ←** `Merge`
  - **Output →** `Revenue Spike Threshold Check`
- **Edge cases / failures:**
  - Key collisions: if two inputs had the same key, later items overwrite earlier ones (not an issue with current distinct keys).

#### Node: Revenue Spike Threshold Check
- **Type / role:** IF node — decides which Slack message to send.
- **Configuration (interpreted):**
  - Condition: `revenue_last_24h >= 100000`
  - Uses expression: `{{ $json.revenue_last_24h }}`
- **Connections:**
  - **Input ←** `Format Sales Data`
  - **True output →** `Send Slack Sales Spike Alert`
  - **False output →** `Progress Alert (Target Pending)`
- **Edge cases / failures:**
  - If `revenue_last_24h` is missing or not numeric, strict validation may fail or condition may behave unexpectedly.
  - Threshold is hardcoded; consider moving to an environment variable or workflow static data if you want easy changes.

#### Node: Send Slack Sales Spike Alert
- **Type / role:** Slack node — posts a celebratory message to a channel if threshold is met.
- **Configuration (interpreted):**
  - Operation: Send message to a **channel** (selected by `channelId`)
  - Message text uses expressions:
    - Revenue, order count, average order value
    - Renders top products:
      - `{{ $json.top_products.map(p => `• ${p[0]} — ${p[1]} unit(s)`).join('\n') }}`
    - Cancellation metrics included
  - Channel: `C09S57E2JQ2` (cached name: “n8n”)
- **Connections:**
  - **Input ←** `Revenue Spike Threshold Check` (true branch)
  - **No downstream outputs**
- **Credentials:** Slack API credential required (bot token with `chat:write` to the channel).
- **Edge cases / failures:**
  - If `top_products` is undefined (merge/format issue), `.map` will throw and message rendering fails.
  - Slack formatting: message contains emoji and markdown-like formatting; Slack generally supports this, but some workspaces restrict formatting.
  - Channel access: bot must be a member of the channel.

#### Node: Progress Alert (Target Pending)
- **Type / role:** Slack node — posts a status message when threshold not reached.
- **Configuration (interpreted):**
  - Similar to spike alert but includes target line and “suggestions”.
  - Channel: `C09S57E2JQ2`
- **Connections:**
  - **Input ←** `Revenue Spike Threshold Check` (false branch)
  - **No downstream outputs**
- **Credentials / edge cases:** Same as the spike alert node.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Daily Revenue Spike Monitor | Schedule Trigger | Daily entry point | — | Fetch WooCommerce Orders | ## Daily Revenue Check<br>Starts the workflow automatically each day. It ensures the revenue check runs on schedule without any manual action and prepares the system to review daily sales performance consistently. |
| Fetch WooCommerce Orders | WooCommerce | Retrieve all orders via API | Daily Revenue Spike Monitor | Filter Paid Orders (Completed / Processing);  Filter Paid Orders (Completed / Processing) | ## Fetch & Filter Orders<br>This section fetches orders from WooCommerce, keeps only paid orders, and filters them to include orders from the last 24 hours. This helps ensure that only valid and recent sales data is used for revenue calculation. |
| Filter Paid Orders (Completed / Processing) | Filter | Keep paid statuses (Processing/Completed) | Fetch WooCommerce Orders | Filter Orders from Last 24 Hours | ## Fetch & Filter Orders<br>This section fetches orders from WooCommerce, keeps only paid orders, and filters them to include orders from the last 24 hours. This helps ensure that only valid and recent sales data is used for revenue calculation. |
| Filter Orders from Last 24 Hours | Code | Filter paid orders by 24h window | Filter Paid Orders (Completed / Processing) | Calculate Top Products (24h); Calculate Total Revenue (24h) | ## Fetch & Filter Orders<br>This section fetches orders from WooCommerce, keeps only paid orders, and filters them to include orders from the last 24 hours. This helps ensure that only valid and recent sales data is used for revenue calculation. |
| Calculate Top Products (24h) | Code | Compute top 3 products by quantity | Filter Orders from Last 24 Hours | Merge | ## 24-Hour Sales Summary<br>This section calculates the total revenue and top-selling products from the last 24 hours. It then merges and formats the data into a single object, providing a clear snapshot of recent sales performance for reporting and alerts. |
| Calculate Total Revenue (24h) | Code | Compute 24h revenue, count, AOV | Filter Orders from Last 24 Hours | Merge | ## 24-Hour Sales Summary<br>This section calculates the total revenue and top-selling products from the last 24 hours. It then merges and formats the data into a single object, providing a clear snapshot of recent sales performance for reporting and alerts. |
|  Filter Paid Orders (Completed / Processing) | Filter | Keep cancelled orders (despite name) | Fetch WooCommerce Orders | Filter Orders from Last 24 Hours2 | ## Cancellation Monitoring (Last 24 Hours)<br>This section tracks cancelled WooCommerce orders from the last 24 hours to measure revenue loss and order drop-offs. It calculates total cancelled orders and associated revenue, helping teams identify potential issues such as payment failures, delivery concerns, or customer experience problems that may impact overall sales performance. |
| Filter Orders from Last 24 Hours2 | Code | Filter cancelled orders by 24h window |  Filter Paid Orders (Completed / Processing) | Calculate Cancelled Metrics (24h) | ## Cancellation Monitoring (Last 24 Hours)<br>This section tracks cancelled WooCommerce orders from the last 24 hours to measure revenue loss and order drop-offs. It calculates total cancelled orders and associated revenue, helping teams identify potential issues such as payment failures, delivery concerns, or customer experience problems that may impact overall sales performance. |
| Calculate Cancelled Metrics (24h) | Code | Compute cancelled count and cancelled revenue | Filter Orders from Last 24 Hours2 | Merge | ## Cancellation Monitoring (Last 24 Hours)<br>This section tracks cancelled WooCommerce orders from the last 24 hours to measure revenue loss and order drop-offs. It calculates total cancelled orders and associated revenue, helping teams identify potential issues such as payment failures, delivery concerns, or customer experience problems that may impact overall sales performance. |
| Merge | Merge | Combine top products + revenue + cancellations | Calculate Top Products (24h); Calculate Total Revenue (24h); Calculate Cancelled Metrics (24h) | Format Sales Data | ## 24-Hour Sales Summary<br>This section calculates the total revenue and top-selling products from the last 24 hours. It then merges and formats the data into a single object, providing a clear snapshot of recent sales performance for reporting and alerts. |
| Format Sales Data | Code | Flatten merged objects to one payload | Merge | Revenue Spike Threshold Check | ## 24-Hour Sales Summary<br>This section calculates the total revenue and top-selling products from the last 24 hours. It then merges and formats the data into a single object, providing a clear snapshot of recent sales performance for reporting and alerts. |
| Revenue Spike Threshold Check | IF | Branch based on revenue threshold | Format Sales Data | Send Slack Sales Spike Alert (true); Progress Alert (Target Pending) (false) | ## Revenue Threshold & Alerts<br>This section checks whether the total revenue from the last 24 hours has reached the set threshold. It then sends Slack notifications: one to celebrate a sales spike and another to provide a status update if the target hasn’t been met, keeping the team informed. |
| Send Slack Sales Spike Alert | Slack | Post spike alert to Slack | Revenue Spike Threshold Check (true) | — | ## Revenue Threshold & Alerts<br>This section checks whether the total revenue from the last 24 hours has reached the set threshold. It then sends Slack notifications: one to celebrate a sales spike and another to provide a status update if the target hasn’t been met, keeping the team informed. |
| Progress Alert (Target Pending) | Slack | Post progress update to Slack | Revenue Spike Threshold Check (false) | — | ## Revenue Threshold & Alerts<br>This section checks whether the total revenue from the last 24 hours has reached the set threshold. It then sends Slack notifications: one to celebrate a sales spike and another to provide a status update if the target hasn’t been met, keeping the team informed. |
| Sticky Note | Sticky Note | Comment block | — | — | ## Daily Revenue Check<br>Starts the workflow automatically each day. It ensures the revenue check runs on schedule without any manual action and prepares the system to review daily sales performance consistently. |
| Sticky Note1 | Sticky Note | Comment block | — | — | ## Fetch & Filter Orders<br>This section fetches orders from WooCommerce, keeps only paid orders, and filters them to include orders from the last 24 hours. This helps ensure that only valid and recent sales data is used for revenue calculation. |
| Sticky Note2 | Sticky Note | Comment block | — | — | ## Revenue Threshold & Alerts<br>This section checks whether the total revenue from the last 24 hours has reached the set threshold. It then sends Slack notifications: one to celebrate a sales spike and another to provide a status update if the target hasn’t been met, keeping the team informed. |
| Sticky Note3 | Sticky Note | Comment block | — | — | ## How This Workflow Works<br>The workflow runs automatically once every day using a Schedule Trigger.<br><br>It fetches recent orders from WooCommerce via the API.<br><br>Only paid orders (Completed / Processing) are considered for revenue calculations to ensure accurate sales data.<br><br>Orders created within the last 24 hours are filtered using a Code node.<br><br>The workflow calculates total revenue, order count, average order value, and identifies the top-selling products for the last 24 hours.<br><br>In parallel, cancelled orders are fetched, filtered for the last 24 hours, and analyzed to calculate cancelled order count and cancelled revenue.<br><br>Sales data and cancellation metrics are merged and formatted into a single, structured object.<br><br>An IF node checks whether the total revenue has crossed the predefined sales threshold.<br><br>If the threshold is crossed, a Slack Sales Spike Alert is sent, including revenue, top products, and cancellation impact.<br><br>If the threshold is not reached, a Slack Status / Pending Alert is sent, showing current performance, top products, and cancellation insights to keep the team informed.<br><br>## How to Set Up This Workflow<br><br>Add a Schedule Trigger to run the workflow daily.<br><br>Add a WooCommerce node to fetch all recent orders.<br><br>Filter orders by Completed / Processing status.<br><br>Use a Code node to filter orders from the last 24 hours.<br><br>Add Code nodes to calculate total revenue, order count, average order value, and top-selling products.<br><br>Add a separate WooCommerce branch to fetch Cancelled orders.<br><br>Filter cancelled orders and limit them to the last 24 hours using a Code node.<br><br>Add a Code node to calculate cancelled order count and cancelled revenue.<br><br>Use a Merge node to combine sales data and cancellation metrics into a single object.<br><br>Add an IF node to compare revenue against the defined threshold.<br><br>Connect a Slack node to send a Sales Spike Alert when the threshold is met.<br><br>Connect another Slack node to send a Target Not Reached / Pending Alert when the threshold is not met.<br><br>Test the workflow and activate it. |
| Sticky Note4 | Sticky Note | Comment block | — | — | ## 24-Hour Sales Summary<br>This section calculates the total revenue and top-selling products from the last 24 hours. It then merges and formats the data into a single object, providing a clear snapshot of recent sales performance for reporting and alerts. |
| Sticky Note5 | Sticky Note | Comment block | — | — | ## Cancellation Monitoring (Last 24 Hours)<br>This section tracks cancelled WooCommerce orders from the last 24 hours to measure revenue loss and order drop-offs. It calculates total cancelled orders and associated revenue, helping teams identify potential issues such as payment failures, delivery concerns, or customer experience problems that may impact overall sales performance. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it: **“WooCommerce Daily Revenue Spike Monitor → Slack Alert”**
- (Optional) Ensure workflow timezone matches your business reporting timezone.

2) **Add trigger: Schedule**
- Add node: **Schedule Trigger**
- Configure: run **daily at 10:00**
- Name: `Daily Revenue Spike Monitor`

3) **Add WooCommerce: Fetch orders**
- Add node: **WooCommerce**
- Credentials: create/select **WooCommerce API** credential:
  - Base URL of store
  - Consumer Key / Consumer Secret (with read access to orders)
- Resource: **Order**
- Operation: **Get All**
- Return All: **true**
- Name: `Fetch WooCommerce Orders`
- Connect: `Daily Revenue Spike Monitor` → `Fetch WooCommerce Orders`

4) **Sales branch: filter paid statuses**
- Add node: **Filter**
- Name: `Filter Paid Orders (Completed / Processing)`
- Condition group: **OR**
  - `{{ $json.status }}` equals `"Processing"`
  - `{{ $json.status }}` equals `"completed"`
- Connect: `Fetch WooCommerce Orders` → `Filter Paid Orders (Completed / Processing)`
  - (Keep the WooCommerce output also available for the cancellation branch later.)

5) **Sales branch: filter last 24 hours**
- Add node: **Code**
- Name: `Filter Orders from Last 24 Hours`
- Paste JS logic:
  - Compute `cutoff = now - 24h`
  - Filter items where `new Date(item.json.date_created) >= cutoff`
- Connect: `Filter Paid Orders (Completed / Processing)` → `Filter Orders from Last 24 Hours`

6) **Sales branch: calculate top products**
- Add node: **Code**
- Name: `Calculate Top Products (24h)`
- Logic:
  - Aggregate `line_items[].quantity` by `line_items[].name`
  - Sort desc, keep top 3
  - Output one item: `{ top_products: [ [name, qty], ... ] }`
- Connect: `Filter Orders from Last 24 Hours` → `Calculate Top Products (24h)`

7) **Sales branch: calculate revenue metrics**
- Add node: **Code**
- Name: `Calculate Total Revenue (24h)`
- Logic:
  - Sum `parseFloat(order.total || 0)`
  - `order_count = items.length`
  - `avg_order_value = revenue / count`
  - Output one item with `revenue_last_24h`, `currency`, `order_count`, `avg_order_value`
- Connect: `Filter Orders from Last 24 Hours` → `Calculate Total Revenue (24h)`

8) **Cancellation branch: filter cancelled**
- Add node: **Filter**
- Name (recommended): `Filter Cancelled Orders` (the provided workflow name contains a leading space; avoid that)
- Condition group: AND (single condition)
  - `{{ $json.status }}` equals `"cancelled"`
- Connect: `Fetch WooCommerce Orders` → `Filter Cancelled Orders`

9) **Cancellation branch: filter last 24 hours**
- Add node: **Code**
- Name: `Filter Orders from Last 24 Hours2`
- Same “last 24 hours” filter code using `date_created`
- Connect: `Filter Cancelled Orders` → `Filter Orders from Last 24 Hours2`

10) **Cancellation branch: compute cancelled KPIs**
- Add node: **Code**
- Name: `Calculate Cancelled Metrics (24h)`
- Logic:
  - `cancelled_orders_24h = items.length`
  - `cancelled_revenue_24h = sum(parseFloat(total || 0))`
  - Output one item with those two fields
- Connect: `Filter Orders from Last 24 Hours2` → `Calculate Cancelled Metrics (24h)`

11) **Merge the three analytics outputs**
- Add node: **Merge**
- Name: `Merge`
- Mode: **Combine**
- Combine by: **Position**
- Number of inputs: **3**
- Connect:
  - `Calculate Top Products (24h)` → `Merge` (input 1 / index 0 in JSON)
  - `Calculate Total Revenue (24h)` → `Merge` (input 2 / index 1)
  - `Calculate Cancelled Metrics (24h)` → `Merge` (input 3 / index 2)

12) **Format into a single object**
- Add node: **Code**
- Name: `Format Sales Data`
- Logic: shallow-merge all incoming items into one object and return one item
- Connect: `Merge` → `Format Sales Data`

13) **Add threshold decision**
- Add node: **IF**
- Name: `Revenue Spike Threshold Check`
- Condition:
  - Value 1: `{{ $json.revenue_last_24h }}`
  - Operator: **greater or equal**
  - Value 2: `100000`
- Connect: `Format Sales Data` → `Revenue Spike Threshold Check`

14) **Slack: configure credentials**
- Create/select **Slack API** credential (bot token).
- Ensure the bot is invited to the target channel and has permission to post.

15) **Slack node (true branch): spike alert**
- Add node: **Slack**
- Name: `Send Slack Sales Spike Alert`
- Action: send message to channel
- Channel: choose your channel (in the JSON: `C09S57E2JQ2`, “n8n”)
- Message text: include variables:
  - `{{ $json.revenue_last_24h }}`, `{{ $json.order_count }}`, `{{ $json.avg_order_value }}`
  - Top products rendering:
    - `{{ $json.top_products.map(p => `• ${p[0]} — ${p[1]} unit(s)`).join('\n') }}`
  - `{{ $json.cancelled_orders_24h }}`, `{{ $json.cancelled_revenue_24h }}`
- Connect: IF **true** → `Send Slack Sales Spike Alert`

16) **Slack node (false branch): progress update**
- Add node: **Slack**
- Name: `Progress Alert (Target Pending)`
- Configure similarly, including the target line (100,000).
- Connect: IF **false** → `Progress Alert (Target Pending)`

17) **Add sticky notes (optional, documentation only)**
- Add sticky notes with the same block headings/content if you want the same layout and maintainability.

18) **Test & activate**
- Execute workflow manually once (with recent order data available).
- Validate Slack messages render correctly (especially top products list).
- Activate workflow.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques. | Compliance / provenance disclaimer (provided by user) |
| Hardcoded currency symbol “₹” is used in Slack messages. Consider switching to `{{ $json.currency }}` or a mapping if you support multi-currency stores. | Slack message formatting consideration |
| Fetching *all* orders and filtering in n8n can be expensive for high-volume stores; prefer API-level filters (date range/status) if performance becomes an issue. | Scaling / performance consideration |
| Status filtering likely needs normalization: WooCommerce commonly uses lowercase statuses (`processing`, `completed`). | Data correctness consideration |