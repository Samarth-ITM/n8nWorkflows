Monitor WooCommerce inventory daily and send Slack alerts with Supabase

https://n8nworkflows.xyz/workflows/monitor-woocommerce-inventory-daily-and-send-slack-alerts-with-supabase-14192


# Monitor WooCommerce inventory daily and send Slack alerts with Supabase

# 1. Workflow Overview

This workflow performs a **daily WooCommerce inventory health check**, estimates stock risk using the last 30 days of completed-order demand, sends **Slack alerts for urgent items**, stores inventory snapshots in **Supabase**, and posts a **daily inventory summary** to Slack.

Its main use cases are:

- Detect products likely to run out soon
- Prioritize reorders using demand-based calculations
- Keep a history of inventory states in a database
- Notify operations teams automatically without manual review

The workflow is organized into the following logical blocks.

## 1.1 Scheduled Execution

The workflow starts automatically once per day at a configured hour.

## 1.2 Product and Sales Collection

It retrieves:
- published WooCommerce products
- completed WooCommerce orders from the last 30 days

Then it calculates per-product sales velocity.

## 1.3 Inventory Enrichment and Classification

It merges product stock data with sales demand, identifies whether stock is managed, estimates days of stock remaining, and classifies products into statuses such as **At Risk**, **Steady**, **Top Performer**, or **Consider Discontinue**.

## 1.4 Reorder Planning and Alert Decision

It calculates safety stock, reorder point, target stock, reorder quantity, then assigns an urgency level and decides whether Slack alerts are required.

## 1.5 Persistence and Reporting

It stores inventory records in Supabase, builds a daily summary from saved rows, and sends a Slack summary message.

## 1.6 Immediate Alerting

For products with **High** or **Critical** urgency, it sends an instant Slack alert with product-level details.

---

# 2. Block-by-Block Analysis

## 2.1 Scheduled Execution

### Overview
This block triggers the workflow automatically every day. It is the sole entry point and fans out into product retrieval and order retrieval in parallel.

### Nodes Involved
- **Run Daily Invetory check**

### Node Details

#### Run Daily Invetory check
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`  
  Triggers the workflow on a schedule.
- **Configuration choices:**  
  Configured with an interval rule that triggers at **hour 2** each day.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - No input
  - Outputs to:
    - **Fecth Products**
    - **Fetch order**
- **Version-specific requirements:**  
  Uses `typeVersion: 1.2`. Schedule UI and recurrence options may differ slightly across n8n versions.
- **Edge cases or potential failure types:**  
  - Workflow must be **active** to run automatically
  - Server timezone affects actual execution time
  - DST changes may shift the effective local hour
- **Sub-workflow reference:**  
  None

---

## 2.2 Product and Sales Collection

### Overview
This block collects all published WooCommerce products and all completed orders from the last 30 days, then calculates average daily demand per product from recent sales.

### Nodes Involved
- **Fecth Products**
- **Fetch order**
- **Calculate product sales**
- **Combine products & sales**

### Node Details

#### Fecth Products
- **Type and technical role:** `n8n-nodes-base.wooCommerce`  
  Retrieves WooCommerce products.
- **Configuration choices:**  
  - Operation: **Get All**
  - Resource: default product resource
  - Return all items: enabled
  - Filters products by status = **publish**
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: **Run Daily Invetory check**
  - Output: **Combine products & sales** input 0
- **Version-specific requirements:**  
  Uses `typeVersion: 1`.
- **Edge cases or potential failure types:**  
  - WooCommerce credential/authentication failure
  - API pagination or large catalog performance issues
  - Missing stock fields on some products
  - Variations or non-stock-managed products may behave differently
- **Sub-workflow reference:**  
  None

#### Fetch order
- **Type and technical role:** `n8n-nodes-base.wooCommerce`  
  Retrieves completed WooCommerce orders.
- **Configuration choices:**  
  - Resource: **order**
  - Operation: **Get All**
  - Return all items: enabled
  - Status filter: **completed**
  - `after` filter set dynamically to 30 days before current time
- **Key expressions or variables used:**  
  - `{{$now.minus({ days: 30 }).toISO()}}`
- **Input and output connections:**  
  - Input: **Run Daily Invetory check**
  - Output: **Calculate product sales**
- **Version-specific requirements:**  
  Uses `typeVersion: 1`.
- **Edge cases or potential failure types:**  
  - Credential/authentication problems
  - Date filtering depends on WooCommerce API behavior and timezone
  - Large order history can increase runtime
  - Orders missing `date_completed` are ignored later
- **Sub-workflow reference:**  
  None

#### Calculate product sales
- **Type and technical role:** `n8n-nodes-base.code`  
  Aggregates units sold per product across the last 30 days and computes average daily demand.
- **Configuration choices:**  
  The JavaScript:
  - Initializes a `productSales` map
  - Calculates `thirtyDaysAgo`
  - Iterates through orders
  - Ignores orders with no `date_completed` or older than 30 days
  - Sums `line_items[].quantity` by `product_id`
  - Outputs:
    - `product_id`
    - `units_sold_last_30_days`
    - `avg_daily_demand = units_sold_last_30_days / 30`
- **Key expressions or variables used:**  
  Internal JS variables:
  - `productSales`
  - `date_completed`
  - `thirtyDaysAgo`
  - `line_items`
- **Input and output connections:**  
  - Input: **Fetch order**
  - Output: **Combine products & sales** input 1
- **Version-specific requirements:**  
  Uses `typeVersion: 2`, meaning Code node execution requires a compatible n8n version with JavaScript Code node support.
- **Edge cases or potential failure types:**  
  - `line_items` missing or empty
  - Orders outside 30-day window produce no output for a product
  - Products with zero sales do not appear here; handled later in merge logic
  - Date parsing inconsistencies if WooCommerce returns malformed timestamps
- **Sub-workflow reference:**  
  None

#### Combine products & sales
- **Type and technical role:** `n8n-nodes-base.merge`  
  Brings product items and sales items into one combined item stream for downstream matching.
- **Configuration choices:**  
  Default merge configuration is used. In this workflow it is functioning as a stream combiner before a custom code join.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input 0: **Fecth Products**
  - Input 1: **Calculate product sales**
  - Output: **Attach sales data to products**
- **Version-specific requirements:**  
  Uses `typeVersion: 3.2`.
- **Edge cases or potential failure types:**  
  - Merge behavior depends on node mode defaults in the n8n version used
  - If the merge mode is changed later, downstream code may break because it expects a mixed stream of product and sales records
- **Sub-workflow reference:**  
  None

---

## 2.3 Inventory Enrichment and Classification

### Overview
This block attaches recent sales demand to each WooCommerce product, computes days of stock left, and assigns a business-oriented inventory status and action recommendation.

### Nodes Involved
- **Attach sales data to products**
- **Inventory Classification**

### Node Details

#### Attach sales data to products
- **Type and technical role:** `n8n-nodes-base.code`  
  Joins WooCommerce product records to the calculated demand data using product ID.
- **Configuration choices:**  
  The JavaScript:
  - Separates incoming mixed items into:
    - `products`
    - `salesMap`
  - Detects products by presence of `stock_quantity` and `id`
  - Detects sales rows by presence of `product_id`
  - For each product, attaches either matching sales data or defaults:
    - `units_sold_last_30_days: 0`
    - `avg_daily_demand: 0`
  - Outputs:
    - `product_id`
    - `sku`
    - `product_name`
    - `stock_quantity`
    - `manage_stock`
    - `units_sold_last_30_days`
    - `avg_daily_demand`
- **Key expressions or variables used:**  
  Internal JS variables:
  - `products`
  - `salesMap`
  - `product.id`
- **Input and output connections:**  
  - Input: **Combine products & sales**
  - Output: **Inventory Classification**
- **Version-specific requirements:**  
  Uses `typeVersion: 2`.
- **Edge cases or potential failure types:**  
  - Products with `stock_quantity = null` may still flow through and affect later math
  - The product detection logic requires `stock_quantity !== undefined`; some WooCommerce product types may not expose it consistently
  - If merge output shape changes, this join logic can fail silently
- **Sub-workflow reference:**  
  None

#### Inventory Classification
- **Type and technical role:** `n8n-nodes-base.code`  
  Classifies inventory health and determines whether reorder consideration is needed.
- **Configuration choices:**  
  The JavaScript:
  - Extracts:
    - `product_id`
    - `sku`
    - `product_name`
    - `stock_quantity`
    - `avg_daily_demand`
    - `manage_stock`
  - Special-cases products where `manage_stock` is false:
    - `inventory_status = 'Not Managed'`
    - `reorder_needed = false`
    - `recommended_action = 'Stock not managed'`
    - `days_of_stock_left = null`
  - Otherwise computes:
    - `days_of_stock_left = stock_quantity / avg_daily_demand` if demand > 0
  - Assigns statuses:
    - Demand = 0 and stock > 0 → **Consider Discontinue**
    - Days < 15 → **At Risk**
    - Days <= 45 → **Steady**
    - Else → **Top Performer**
  - Sets recommended action accordingly
- **Key expressions or variables used:**  
  Internal JS variables:
  - `days_of_stock_left`
  - `inventory_status`
  - `reorder_needed`
  - `recommended_action`
- **Input and output connections:**  
  - Input: **Attach sales data to products**
  - Output: **Calculate reorder quantity**
- **Version-specific requirements:**  
  Uses `typeVersion: 2`.
- **Edge cases or potential failure types:**  
  - If `stock_quantity` is null, divisions and comparisons can produce misleading results
  - If `avg_daily_demand = 0` and `stock_quantity <= 0`, the logic falls through to `Top Performer`, which is likely unintended
  - Negative stock values are not explicitly handled
  - Non-managed stock items still continue downstream into reorder calculations
- **Sub-workflow reference:**  
  None

---

## 2.4 Reorder Planning and Alert Decision

### Overview
This block converts inventory classification into actionable planning metrics such as reorder point and reorder quantity, then determines urgency and whether a Slack alert should be sent.

### Nodes Involved
- **Calculate reorder quantity**
- **Set urgency & action plan**
- **Check if alerts is needed**

### Node Details

#### Calculate reorder quantity
- **Type and technical role:** `n8n-nodes-base.code`  
  Computes safety stock, reorder point, target stock, and suggested reorder quantity.
- **Configuration choices:**  
  Hard-coded planning constants:
  - `LEAD_TIME_DAYS = 7`
  - `SAFETY_DAYS = 5`
  
  Formula logic:
  - `dailyDemand = max(avg_daily_demand, 0.01)`
  - `safety_stock = dailyDemand * 5`
  - `reorder_point = dailyDemand * 7 + safety_stock`
  - `target_stock = dailyDemand * (7 + 5)`
  - `reorder_qty = ceil(target_stock - stock_quantity)`, minimum 0
- **Key expressions or variables used:**  
  Internal JS variables:
  - `LEAD_TIME_DAYS`
  - `SAFETY_DAYS`
  - `dailyDemand`
  - `safety_stock`
  - `reorder_point`
  - `target_stock`
  - `reorder_qty`
- **Input and output connections:**  
  - Input: **Inventory Classification**
  - Output: **Set urgency & action plan**
- **Version-specific requirements:**  
  Uses `typeVersion: 2`.
- **Edge cases or potential failure types:**  
  - `avg_daily_demand` is forced to at least `0.01`, so even zero-demand products may get non-zero planning values
  - Non-managed products continue through and can receive calculated reorder metrics even though reorder is not needed
  - Null stock values can distort reorder quantity
- **Sub-workflow reference:**  
  None

#### Set urgency & action plan
- **Type and technical role:** `n8n-nodes-base.code`  
  Assigns urgency level and a final human-readable action based on reorder quantity and days of stock left.
- **Configuration choices:**  
  Logic:
  - Default urgency: **Normal**
  - Default action: previous `recommended_action`
  - If `reorder_qty > 0`:
    - `days_of_stock_left <= 7` → **Critical**
    - `days_of_stock_left <= 14` → **High**
    - otherwise → **Normal**
  - Final action strings include reorder quantity
- **Key expressions or variables used:**  
  Internal JS variables:
  - `urgency`
  - `action`
  - `d.reorder_qty`
  - `d.days_of_stock_left`
- **Input and output connections:**  
  - Input: **Calculate reorder quantity**
  - Output: **Check if alerts is needed**
- **Version-specific requirements:**  
  Uses `typeVersion: 2`.
- **Edge cases or potential failure types:**  
  - If `days_of_stock_left` is `null`, comparisons against 7 or 14 are false, so urgency remains **Normal**
  - Zero-demand items can still have reorder quantity due to minimum demand floor, which may produce misleading actions
  - Not-managed items may remain Normal but still carry reorder calculations
- **Sub-workflow reference:**  
  None

#### Check if alerts is needed
- **Type and technical role:** `n8n-nodes-base.if`  
  Routes urgent products to alerting and non-urgent products to persistence only.
- **Configuration choices:**  
  OR condition:
  - `urgency_level == "Critical"`
  - `urgency_level == "High"`
  
  In n8n IF nodes:
  - **true output** goes to alerting
  - **false output** goes to database persistence
- **Key expressions or variables used:**  
  - `={{ $json.urgency_level }}`
- **Input and output connections:**  
  - Input: **Set urgency & action plan**
  - True output → **Send critical inventory alert**
  - False output → **Save Inventory record to database**
- **Version-specific requirements:**  
  Uses `typeVersion: 2.2`.
- **Edge cases or potential failure types:**  
  - The node name is grammatically incorrect but functional
  - Only false-branch items go to database saving in the current design
  - High/Critical items sent to Slack are **not** persisted unless an additional connection is added
- **Sub-workflow reference:**  
  None

---

## 2.5 Immediate Alerting

### Overview
This block sends Slack alerts for products that require urgent action, specifically items classified as High or Critical urgency.

### Nodes Involved
- **Send critical inventory alert**

### Node Details

#### Send critical inventory alert
- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends a formatted message to a Slack channel.
- **Configuration choices:**  
  - Sends a plain text message to a selected Slack channel
  - Channel selected by ID `C09S57E2JQ2`
  - Workflow link inclusion disabled
  - Message includes:
    - product name
    - SKU
    - inventory status
    - stock quantity
    - final recommended action
    - urgency
    - reorder quantity
- **Key expressions or variables used:**  
  - `{{ $json.product_name }}`
  - `{{ $json.sku }}`
  - `{{ $json.inventory_status }}`
  - `{{ $json.stock_quantity }}`
  - `{{ $json.final_recommended_action }}`
  - `{{ $json.urgency_level }}`
  - `{{ $json.reorder_qty }}`
- **Input and output connections:**  
  - Input: **Check if alerts is needed** true branch
  - No downstream output
- **Version-specific requirements:**  
  Uses `typeVersion: 2.3`.
- **Edge cases or potential failure types:**  
  - Slack credential issues or revoked token
  - Invalid or inaccessible channel ID
  - Message formatting is simple text; `** Inventory Alert **` is not standard Slack markdown emphasis
  - Because there is no downstream connection, alerted items stop here and are not summarized or saved
- **Sub-workflow reference:**  
  None

---

## 2.6 Persistence and Reporting

### Overview
This block stores non-urgent inventory records in Supabase, converts stored rows into a daily summary, and posts the summary to Slack.

### Nodes Involved
- **Save Inventory record to database**
- **Prepare Inventory summary**
- **Send Inventory summry**

### Node Details

#### Save Inventory record to database
- **Type and technical role:** `n8n-nodes-base.supabase`  
  Inserts inventory records into a Supabase table.
- **Configuration choices:**  
  - Table: **Inventory Data**
  - Fields mapped:
    - `product id` ← `{{$json.product_id}}`
    - `product name` ← `{{$json.product_name}}`
    - `inventory status` ← `{{$json.inventory_status}}`
    - `recommended action` ← `{{$json.recommended_action}}`
    - `quantity` ← `{{$json.stock_quantity}}`
- **Key expressions or variables used:**  
  - `={{ $json.product_id }}`
  - `={{ $json.product_name }}`
  - `={{ $json.inventory_status }}`
  - `={{ $json.recommended_action }}`
  - `={{ $json.stock_quantity }}`
- **Input and output connections:**  
  - Input: **Check if alerts is needed** false branch
  - Output: **Prepare Inventory summary**
- **Version-specific requirements:**  
  Uses `typeVersion: 1`.
- **Edge cases or potential failure types:**  
  - Supabase auth or row-level security restrictions
  - Field names include spaces; the table schema must match exactly
  - It stores `recommended_action`, not `final_recommended_action`
  - High/Critical items are excluded from storage due to branch design
- **Sub-workflow reference:**  
  None

#### Prepare Inventory summary
- **Type and technical role:** `n8n-nodes-base.code`  
  Aggregates the saved inventory rows into a one-item summary for Slack.
- **Configuration choices:**  
  The JavaScript:
  - Counts total items
  - Counts rows where `"inventory status" === "At Risk"`
  - Uses the same condition to count `reorderNeeded`
  - Builds bullet-point summary lines:
    - `product name`
    - `inventory status`
    - `quantity`
  - Returns one JSON object with:
    - `totalProducts`
    - `atRisk`
    - `reorderNeeded`
    - `summaryText`
- **Key expressions or variables used:**  
  Internal JS field access:
  - `d["inventory status"]`
  - `d["product name"]`
  - `d.quantity`
- **Input and output connections:**  
  - Input: **Save Inventory record to database**
  - Output: **Send Inventory summry**
- **Version-specific requirements:**  
  Uses `typeVersion: 2`.
- **Edge cases or potential failure types:**  
  - Summary is based only on rows returned from the Supabase node, not the original full inventory set
  - Because only false-branch items are saved, the summary omits High/Critical products
  - `reorderNeeded` is inferred from status, not an actual stored reorder flag
- **Sub-workflow reference:**  
  None

#### Send Inventory summry
- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends a daily summary message to Slack.
- **Configuration choices:**  
  - Sends to the same Slack channel ID `C09S57E2JQ2`
  - Workflow link inclusion disabled
  - Message contains:
    - total checked products
    - at-risk products
    - reorder needed count
    - product overview list
- **Key expressions or variables used:**  
  - `{{ $json.totalProducts }}`
  - `{{ $json.atRisk }}`
  - `{{ $json.reorderNeeded }}`
  - `{{ $json.summaryText }}`
- **Input and output connections:**  
  - Input: **Prepare Inventory summary**
  - No downstream output
- **Version-specific requirements:**  
  Uses `typeVersion: 2.3`.
- **Edge cases or potential failure types:**  
  - Slack auth/channel issues
  - Large summaries can hit Slack message-length or readability limits
  - Node name contains a typo but is functional
- **Sub-workflow reference:**  
  None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Run Daily Invetory check | Schedule Trigger | Starts the workflow daily at the configured time |  | Fecth Products; Fetch order | Runs this inventory check automatically every day at the scheduled time. |
| Fecth Products | WooCommerce | Fetches all published WooCommerce products | Run Daily Invetory check | Combine products & sales  | ## Collect Product & Sales Data<br>This step gathers all products from the store, checks how much stock is available, looks at sales from the last 30 days, and combines everything to understand how fast each product is selling. |
| Fetch order | WooCommerce | Fetches completed WooCommerce orders from the last 30 days | Run Daily Invetory check | Calculate product sales | ## Collect Product & Sales Data<br>This step gathers all products from the store, checks how much stock is available, looks at sales from the last 30 days, and combines everything to understand how fast each product is selling. |
| Calculate product sales | Code | Aggregates product sales and computes average daily demand | Fetch order | Combine products & sales  | ## Collect Product & Sales Data<br>This step gathers all products from the store, checks how much stock is available, looks at sales from the last 30 days, and combines everything to understand how fast each product is selling. |
| Combine products & sales  | Merge | Combines product and sales streams before custom joining | Fecth Products; Calculate product sales | Attach sales data to products | ## Collect Product & Sales Data<br>This step gathers all products from the store, checks how much stock is available, looks at sales from the last 30 days, and combines everything to understand how fast each product is selling. |
| Attach sales data to products | Code | Matches sales metrics to each product | Combine products & sales  | Inventory Classification | ## Inventory Evaluation, Planning & Decision<br>This step combines product and sales data, checks stock health based on demand, categorizes products by performance, calculates safety stock and reorder quantity, assigns urgency levels, and finally decides which products need immediate attention and alerts. |
| Inventory Classification | Code | Classifies product inventory health and recommended action | Attach sales data to products | Calculate reorder quantity | ## Inventory Evaluation, Planning & Decision<br>This step combines product and sales data, checks stock health based on demand, categorizes products by performance, calculates safety stock and reorder quantity, assigns urgency levels, and finally decides which products need immediate attention and alerts. |
| Calculate reorder quantity | Code | Calculates safety stock, reorder point, target stock, and reorder quantity | Inventory Classification | Set urgency & action plan | ## Inventory Evaluation, Planning & Decision<br>This step combines product and sales data, checks stock health based on demand, categorizes products by performance, calculates safety stock and reorder quantity, assigns urgency levels, and finally decides which products need immediate attention and alerts. |
| Set urgency & action plan | Code | Converts reorder needs into urgency and final action text | Calculate reorder quantity | Check if alerts is needed | ## Inventory Evaluation, Planning & Decision<br>This step combines product and sales data, checks stock health based on demand, categorizes products by performance, calculates safety stock and reorder quantity, assigns urgency levels, and finally decides which products need immediate attention and alerts. |
| Check if alerts is needed | If | Routes urgent products to Slack alerts and others to storage | Set urgency & action plan | Send critical inventory alert; Save Inventory record to database | ## Inventory Evaluation, Planning & Decision<br>This step combines product and sales data, checks stock health based on demand, categorizes products by performance, calculates safety stock and reorder quantity, assigns urgency levels, and finally decides which products need immediate attention and alerts. |
| Send critical inventory alert | Slack | Sends urgent product-level Slack alerts | Check if alerts is needed |  | Sends instant Slack alerts for products that require urgent action or immediate reorder. |
| Save Inventory record to database | Supabase | Stores inventory records in Supabase | Check if alerts is needed | Prepare Inventory summary | ## Inventory Record, Summarize & Notify Inventory Status<br><br>This step saves the latest inventory results into the system, creates a daily summary of stock health and risk items, and sends a clear report to the team on Slack so everyone stays informed. |
| Prepare Inventory summary | Code | Builds a one-message inventory summary from stored records | Save Inventory record to database | Send Inventory summry | ## Inventory Record, Summarize & Notify Inventory Status<br><br>This step saves the latest inventory results into the system, creates a daily summary of stock health and risk items, and sends a clear report to the team on Slack so everyone stays informed. |
| Send Inventory summry | Slack | Sends the daily inventory summary to Slack | Prepare Inventory summary |  | ## Inventory Record, Summarize & Notify Inventory Status<br><br>This step saves the latest inventory results into the system, creates a daily summary of stock health and risk items, and sends a clear report to the team on Slack so everyone stays informed. |
| Sticky Note | Sticky Note | Visual documentation for the schedule block |  |  | Runs this inventory check automatically every day at the scheduled time. |
| Sticky Note1 | Sticky Note | Visual documentation for product and sales collection |  |  | ## Collect Product & Sales Data<br>This step gathers all products from the store, checks how much stock is available, looks at sales from the last 30 days, and combines everything to understand how fast each product is selling. |
| Sticky Note2 | Sticky Note | Visual documentation for evaluation and planning |  |  | ## Inventory Evaluation, Planning & Decision<br>This step combines product and sales data, checks stock health based on demand, categorizes products by performance, calculates safety stock and reorder quantity, assigns urgency levels, and finally decides which products need immediate attention and alerts. |
| Sticky Note3 | Sticky Note | Visual documentation for storage and summary reporting |  |  | ## Inventory Record, Summarize & Notify Inventory Status<br><br>This step saves the latest inventory results into the system, creates a daily summary of stock health and risk items, and sends a clear report to the team on Slack so everyone stays informed. |
| Sticky Note4 | Sticky Note | Visual documentation for urgent alerting |  |  | Sends instant Slack alerts for products that require urgent action or immediate reorder. |
| Sticky Note5 | Sticky Note | General workflow explanation and setup guidance |  |  | ## How it works<br><br>This workflow automatically checks your store’s inventory every day and helps you avoid stock issues.<br><br>First, it pulls all active products and recent completed orders from WooCommerce. Using sales from the last 30 days, it calculates how fast each product is selling.<br>Next, it compares current stock with demand to understand how healthy your inventory is.<br><br>Each product is then classified into clear categories like Top Performer, Steady, At Risk, or Consider Discontinue. Based on this, the system calculates how much safety stock is needed, when to reorder, and how many units should be ordered.<br><br>Products that need attention are given an urgency level (Normal, High, or Critical).<br>Important items trigger alerts, while all results are saved for reporting. Finally, a daily inventory summary is sent to Slack so the team can quickly review overall stock health without checking multiple systems.<br><br>This process runs automatically and requires no manual effort once set up.<br><br><br>## Setup steps<br><br>**1.** Connect your WooCommerce account (products and orders).<br><br>**2.** Connect Supabase to store inventory records.<br><br>**3.** Connect Slack to receive alerts and daily summaries.<br><br>**4.** Set the schedule time for daily checks.<br><br>**5.** Review stock thresholds (lead time, safety days) if needed. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.  
   Name it something like **Daily Inventory Monitoring & Reorder System**.

2. **Add a Schedule Trigger node** named **Run Daily Invetory check**.
   - Node type: **Schedule Trigger**
   - Configure it to run **daily at 2:00**
   - Confirm your n8n instance timezone is correct

3. **Add a WooCommerce node** named **Fecth Products**.
   - Node type: **WooCommerce**
   - Credentials: connect a WooCommerce API credential
   - Operation: **Get All**
   - Resource: products
   - Return All: **true**
   - Options → status: **publish**
   - Connect **Run Daily Invetory check → Fecth Products**

4. **Add another WooCommerce node** named **Fetch order**.
   - Node type: **WooCommerce**
   - Credentials: use the same WooCommerce credential or another valid one
   - Resource: **order**
   - Operation: **Get All**
   - Return All: **true**
   - Options:
     - status: **completed**
     - after: `{{$now.minus({ days: 30 }).toISO()}}`
   - Connect **Run Daily Invetory check → Fetch order**

5. **Add a Code node** named **Calculate product sales**.
   - Node type: **Code**
   - Language: JavaScript
   - Paste this logic conceptually:
     - iterate through completed orders
     - ignore orders older than 30 days or without `date_completed`
     - sum line item quantities by `product_id`
     - output `product_id`, `units_sold_last_30_days`, and `avg_daily_demand`
   - Connect **Fetch order → Calculate product sales**

6. **Add a Merge node** named **Combine products & sales**.
   - Node type: **Merge**
   - Keep default behavior consistent with a simple combination of both input streams
   - Connect:
     - **Fecth Products → Combine products & sales** input 1
     - **Calculate product sales → Combine products & sales** input 2

7. **Add a Code node** named **Attach sales data to products**.
   - Purpose:
     - split incoming mixed items into products and sales rows
     - create a sales lookup keyed by `product_id`
     - output one item per product with attached sales metrics
   - Output fields:
     - `product_id`
     - `sku`
     - `product_name`
     - `stock_quantity`
     - `manage_stock`
     - `units_sold_last_30_days`
     - `avg_daily_demand`
   - Default sales values when none found:
     - `units_sold_last_30_days = 0`
     - `avg_daily_demand = 0`
   - Connect **Combine products & sales → Attach sales data to products**

8. **Add a Code node** named **Inventory Classification**.
   - Purpose:
     - handle unmanaged stock items
     - compute days of stock left
     - classify inventory status
   - Implement logic:
     - if `manage_stock` is false:
       - `days_of_stock_left = null`
       - `inventory_status = "Not Managed"`
       - `reorder_needed = false`
       - `recommended_action = "Stock not managed"`
     - else if `avg_daily_demand === 0 && stock_quantity > 0`:
       - `inventory_status = "Consider Discontinue"`
       - `recommended_action = "Stop restocking / apply discount"`
     - else if `days_of_stock_left < 15`:
       - `inventory_status = "At Risk"`
       - `reorder_needed = true`
       - `recommended_action = "Reorder immediately"`
     - else if `days_of_stock_left <= 45`:
       - `inventory_status = "Steady"`
       - `recommended_action = "Monitor stock"`
     - else:
       - `inventory_status = "Top Performer"`
       - `recommended_action = "Maintain stock level"`
   - Connect **Attach sales data to products → Inventory Classification**

9. **Add a Code node** named **Calculate reorder quantity**.
   - Purpose:
     - compute stock planning values
   - Use constants:
     - `LEAD_TIME_DAYS = 7`
     - `SAFETY_DAYS = 5`
   - Compute:
     - `dailyDemand = Math.max(avg_daily_demand || 0, 0.01)`
     - `safety_stock`
     - `reorder_point`
     - `target_stock`
     - `reorder_qty = max(ceil(target_stock - stock_quantity), 0)`
   - Output all previous fields plus:
     - `safety_stock`
     - `reorder_point`
     - `target_stock`
     - `reorder_qty`
   - Connect **Inventory Classification → Calculate reorder quantity**

10. **Add a Code node** named **Set urgency & action plan**.
    - Purpose:
      - map reorder quantity and days of stock to urgency
    - Logic:
      - default `urgency_level = "Normal"`
      - default `final_recommended_action = recommended_action`
      - if `reorder_qty > 0` and `days_of_stock_left <= 7`:
        - urgency = `Critical`
        - action = `Order X units immediately`
      - else if `reorder_qty > 0` and `days_of_stock_left <= 14`:
        - urgency = `High`
        - action = `Plan reorder of X units`
      - else if `reorder_qty > 0`:
        - urgency = `Normal`
        - action = `Monitor – next reorder X units`
    - Connect **Calculate reorder quantity → Set urgency & action plan**

11. **Add an IF node** named **Check if alerts is needed**.
    - Configure OR conditions:
      - `{{$json.urgency_level}}` equals `Critical`
      - `{{$json.urgency_level}}` equals `High`
    - Connect **Set urgency & action plan → Check if alerts is needed**

12. **Add a Slack node** named **Send critical inventory alert**.
    - Credentials: connect a Slack API credential
    - Operation: send message to channel
    - Choose the target channel
    - Disable “include link to workflow” if desired
    - Message text should include:
      - product name
      - SKU
      - status
      - stock
      - final recommended action
      - urgency
      - reorder quantity
    - Example dynamic fields:
      - `{{ $json.product_name }}`
      - `{{ $json.sku }}`
      - `{{ $json.inventory_status }}`
      - `{{ $json.stock_quantity }}`
      - `{{ $json.final_recommended_action }}`
      - `{{ $json.urgency_level }}`
      - `{{ $json.reorder_qty }}`
    - Connect the **true output** of **Check if alerts is needed → Send critical inventory alert**

13. **Add a Supabase node** named **Save Inventory record to database**.
    - Credentials: connect a Supabase API credential
    - Operation: insert/create row
    - Table: **Inventory Data**
    - Make sure the table exists with columns matching the names used in mapping
    - Map fields:
      - `product id` ← `{{$json.product_id}}`
      - `product name` ← `{{$json.product_name}}`
      - `inventory status` ← `{{$json.inventory_status}}`
      - `recommended action` ← `{{$json.recommended_action}}`
      - `quantity` ← `{{$json.stock_quantity}}`
    - Connect the **false output** of **Check if alerts is needed → Save Inventory record to database**

14. **Create the Supabase table before testing**.
    - Required table: **Inventory Data**
    - Required columns, as used by this workflow:
      - `product id`
      - `product name`
      - `inventory status`
      - `recommended action`
      - `quantity`
    - Ensure permissions allow inserts from your credential

15. **Add a Code node** named **Prepare Inventory summary**.
    - Purpose:
      - aggregate database output rows into one daily summary object
    - Logic:
      - count all incoming rows as `totalProducts`
      - count rows where `"inventory status" === "At Risk"` as `atRisk`
      - also count those as `reorderNeeded`
      - build a bullet list:
        - `• product name → inventory status (Stock: quantity)`
      - return one item with:
        - `totalProducts`
        - `atRisk`
        - `reorderNeeded`
        - `summaryText`
    - Connect **Save Inventory record to database → Prepare Inventory summary**

16. **Add a Slack node** named **Send Inventory summry**.
    - Credentials: use Slack API credential
    - Send to the desired channel
    - Message should include:
      - total checked products
      - at risk products
      - reorder needed
      - product status overview
      - summary text
    - Connect **Prepare Inventory summary → Send Inventory summry**

17. **Optionally add sticky notes** for maintainability.
    Suggested notes:
    - schedule explanation
    - data collection explanation
    - planning/decision explanation
    - urgent alert explanation
    - storage/report explanation
    - global “How it works / Setup steps” note

18. **Configure credentials**.
    - **WooCommerce**
      - Base store URL
      - Consumer key
      - Consumer secret
    - **Slack**
      - OAuth or token-based Slack credential with permission to post in the target channel
    - **Supabase**
      - Project URL
      - API key/service role or equivalent allowed by your security model

19. **Test each branch manually**.
    - Run product fetch
    - Run order fetch
    - Confirm sales aggregation output
    - Confirm product-sales join output
    - Confirm classification and reorder fields
    - Confirm IF node routes expected items correctly
    - Confirm Slack delivery and Supabase inserts

20. **Activate the workflow** after validation.

---

## Reproduction Notes and Behavioral Constraints

1. **This workflow has one explicit entry point only**: the Schedule Trigger.

2. **There are no sub-workflows** and no Execute Workflow nodes.

3. **Important current design behavior:**  
   High/Critical items go to Slack only, while non-urgent items go to Supabase and the summary chain.  
   If you want **all products** stored and summarized, connect both IF branches into a common persistence path, or store first and alert second.

4. **Recommended schema caution:**  
   Using Supabase column names with spaces works only if your table is defined exactly that way. For long-term maintainability, consider snake_case column names instead.

5. **Recommended logic improvement:**  
   Review how zero-demand and unmanaged items are handled in reorder calculations, because the current minimum-demand floor (`0.01`) can produce reorder suggestions for products that may not need them.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow automatically checks store inventory daily, calculates recent demand from completed WooCommerce orders, classifies stock health, estimates reorder needs, sends urgent alerts, stores results, and posts a Slack summary. | General workflow description |
| Setup steps listed in the workflow notes: connect WooCommerce, connect Supabase, connect Slack, set the daily schedule, and review lead-time/safety thresholds. | General setup guidance |
| Workflow title provided by the user: **Monitor WooCommerce inventory daily and send Slack alerts with Supabase** | External title / labeling context |
| Internal workflow name in JSON: **Daily Inventory Monitoring & Reorder System** | Workflow metadata |