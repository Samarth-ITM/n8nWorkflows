Sync WooCommerce orders and inventory with Google Sheets and Slack

https://n8nworkflows.xyz/workflows/sync-woocommerce-orders-and-inventory-with-google-sheets-and-slack-11968


# Sync WooCommerce orders and inventory with Google Sheets and Slack

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Sync WooCommerce orders and inventory with Google Sheets and Slack  
**Workflow name (in JSON):** Automate order fulfillment and inventory sync from WooCommerce to Google Sheets and Slack  
**Purpose:** Automatically ingest WooCommerce orders (via webhook and/or scheduled trigger), normalize the order payload, check inventory per line item, decide whether the order is fulfillable, notify the customer and internal team (Slack + Gmail), then log the outcome to Google Sheets and respond to the webhook.

### 1.1 Order Intake (Triggers + Normalization)
- Accepts new orders from WooCommerce via webhook and optionally via a scheduled trigger.
- Merges the two entry points and transforms raw data into a normalized JSON object.

### 1.2 Inventory Check (Per-item + Aggregation)
- Splits order items into one item per execution item.
- Performs an inventory lookup (currently simulated).
- Aggregates per-item results back into one object for order-level decisions.

### 1.3 Processing Logic (Fulfillment decision)
- Builds the final order object: inventory summary, fulfillment status/priority, and a selected shipping rate.
- Branches based on whether everything is in stock.

### 1.4 Automated Notifications (Customer + Team)
- If in stock: send customer confirmation + Slack notification.
- If not in stock: send backorder email + Slack alert.

### 1.5 Logging & Response (Sheets + Webhook response)
- Merges the two notification paths.
- Appends the order outcome to Google Sheets.
- Returns a JSON success response to the webhook caller.

---

## 2. Block-by-Block Analysis

### Block 1 — Order Intake

**Overview:** Receives order events from WooCommerce or a scheduled run, then normalizes the payload into a consistent structure used throughout the workflow.

**Nodes involved:**
- Main Sticky Note (documentation)
- Group Sticky 1 (documentation)
- WooCommerce Order Webhook1
- Scheduled Order Sync1
- Merge Triggers1
- Extract Order Data1

#### Node: **WooCommerce Order Webhook1**
- **Type / role:** `Webhook` — primary real-time entry point for WooCommerce order creation/update events.
- **Configuration choices:**
  - **HTTP Method:** POST
  - **Path:** `woo-new-order` (final URL depends on n8n instance base URL)
  - **Response mode:** “Respond with Respond to Webhook node” (defers response until end)
- **Key variables/expressions:** none
- **Connections:**
  - Output → **Merge Triggers1** (branch index 0)
- **Version requirements:** Webhook node `typeVersion: 2`
- **Potential failures / edge cases:**
  - WooCommerce webhook misconfiguration (wrong URL, secret, event type)
  - Payload differences: `body` may not exist depending on how requests are forwarded; later code attempts `input.body || input`
  - If no later Respond node is executed (errors mid-flow), WooCommerce receives no/late response and may retry

#### Node: **Scheduled Order Sync1**
- **Type / role:** `Schedule Trigger` — periodic entry point for hourly sync.
- **Configuration choices:**
  - Runs every **hour** (`interval: hours`)
  - No built-in WooCommerce fetch is implemented in this workflow; it only triggers the same downstream path.
- **Connections:**
  - Output → **Merge Triggers1** (branch index 1)
- **Version requirements:** `typeVersion: 1.2`
- **Potential failures / edge cases:**
  - As-is, scheduled runs will not include actual order data unless you add a WooCommerce “Get Orders” step before normalization.
  - Downstream code expects order-like fields; scheduled trigger outputs a timestamp payload by default, which will cause missing-field behavior (e.g., undefined `order.id`).

#### Node: **Merge Triggers1**
- **Type / role:** `Merge` (mode: chooseBranch) — allows either trigger path to continue.
- **Configuration choices:**
  - **Mode:** Choose Branch (passes through whichever branch runs)
- **Connections:**
  - Input 1 ← **WooCommerce Order Webhook1**
  - Input 2 ← **Scheduled Order Sync1**
  - Output → **Extract Order Data1**
- **Version requirements:** `typeVersion: 3`
- **Potential failures / edge cases:**
  - If scheduled trigger is used without a fetch/enrichment step, downstream normalization may produce invalid data.

#### Node: **Extract Order Data1**
- **Type / role:** `Code` — transforms raw WooCommerce payload into normalized order JSON.
- **Configuration choices (interpreted):**
  - Reads first input item: `const input = $input.first().json;`
  - Uses `input.body || input` to support webhook payloads (often under `body`)
  - Maps `order.line_items` into `items[]` with:
    - `productId`, `name`, `sku` (fallback `N/A`), `quantity`, `price`, `total`
  - Extracts `shipping` and `billing` blocks
  - Outputs normalized shape:
    - `orderId`, `orderNumber` (fallback `ORD-<timestamp>`), `items`, `customer.email`, `customer.firstName`, `shipping.city`, `shipping.country` (fallback `US`), `total`
- **Key expressions/variables:**
  - Uses n8n Code node APIs: `$input.first()`
- **Connections:**
  - Output → **Split Order Items1**
- **Version requirements:** `typeVersion: 2`
- **Potential failures / edge cases:**
  - Missing `billing.email` → Gmail nodes later will fail or sendTo becomes empty
  - `order.line_items` missing/non-array → `items` becomes `[]`; split node may emit 0 items causing downstream aggregation to behave unexpectedly
  - Numeric parsing: `parseFloat(undefined)` becomes `NaN` but code guards with `|| 0` in some fields; ensure consistent handling if WooCommerce sends empty strings
  - Scheduled trigger path likely lacks `body` and Woo fields → produces `orderId: undefined`

---

### Block 2 — Inventory Check

**Overview:** Splits the order into individual line items, checks stock per item (simulated), then aggregates results for order-level logic.

**Nodes involved:**
- Group Sticky 2 (documentation)
- Split Order Items1
- Check Inventory1
- Aggregate Inventory Results1

#### Node: **Split Order Items1**
- **Type / role:** `Split Out` — converts `items[]` array into one n8n item per array element.
- **Configuration choices:**
  - **Field to split out:** `items`
- **Connections:**
  - Input ← **Extract Order Data1**
  - Output → **Check Inventory1**
- **Version requirements:** `typeVersion: 1`
- **Potential failures / edge cases:**
  - If `items` is empty/missing/not an array → may output zero items; later nodes may not execute as expected
  - If item objects are malformed, inventory logic may not have `productId`/`sku`

#### Node: **Check Inventory1**
- **Type / role:** `Code` — placeholder inventory lookup.
- **Configuration choices (interpreted):**
  - Reads the current split item: `const item = $input.first().json;`
  - Returns the item plus simulated inventory status:
    - `isInStock: true`
    - `stockStatus: 'available'`
- **Connections:**
  - Output → **Aggregate Inventory Results1**
- **Version requirements:** `typeVersion: 2`
- **Potential failures / edge cases:**
  - This is a stub: real implementation should call an inventory DB/ERP/WMS.
  - If replaced with external API calls, handle timeouts, rate limits, partial failures, and per-SKU not found cases.

#### Node: **Aggregate Inventory Results1**
- **Type / role:** `Aggregate` — aggregates all item-level inventory results into one structure.
- **Configuration choices:**
  - **Aggregate mode:** “aggregateAllItemData”
  - Stores aggregated results under `data` (as used later)
- **Connections:**
  - Input ← **Check Inventory1**
  - Output → **Process Order Logic1**
- **Version requirements:** `typeVersion: 1`
- **Potential failures / edge cases:**
  - If zero items enter this node (e.g., empty cart), aggregation may output nothing; downstream code using `$input.first()` will fail.
  - Ensure consistent aggregated schema if you later change aggregate settings.

---

### Block 3 — Processing Logic

**Overview:** Creates an enriched “final order” record (inventory summary, fulfillment decision, and shipping selection) and branches the workflow into “in stock” vs “backorder”.

**Nodes involved:**
- Group Sticky 3 (documentation)
- Process Order Logic1
- Check Fulfillment Status1

#### Node: **Process Order Logic1**
- **Type / role:** `Code` — computes fulfillment state and shipping selection.
- **Configuration choices (interpreted):**
  - Retrieves aggregated inventory results: `const items = $input.first().json.data;`
    - (Note: `items` variable is not used afterward in the current code.)
  - Pulls normalized order data by cross-node reference:
    - `const orderData = $('Extract Order Data1').first().json;`
  - Returns merged object with hard-coded decisions:
    - `inventory.allInStock: true`
    - `fulfillment.status: 'ready'`
    - `fulfillment.priority: 'normal'`
    - `shipping.selectedRate: { carrier: 'UPS', method: 'Ground', days: '3-5' }`
- **Connections:**
  - Output → **Check Fulfillment Status1**
- **Version requirements:** `typeVersion: 2`
- **Potential failures / edge cases:**
  - Cross-node reference `$('Extract Order Data1')...` can fail if that node didn’t run in the same execution path (especially if you later change branching).
  - Inventory summary is currently hard-coded; if you compute real `allInStock`, ensure you handle partial stock and missing SKUs.
  - If aggregation produced no items, `$input.first()` may be undefined.

#### Node: **Check Fulfillment Status1**
- **Type / role:** `IF` — branches based on `inventory.allInStock`.
- **Configuration choices:**
  - Condition: `{{ $json.inventory.allInStock }}` **equals** `true`
- **Connections:**
  - **True** output → **Send Customer Confirmation1** and **Notify Fulfillment Team1** (parallel)
  - **False** output → **Send Backorder Notice1** and **Backorder Alert1** (parallel)
- **Version requirements:** `typeVersion: 2`
- **Potential failures / edge cases:**
  - If `inventory.allInStock` is missing/null/non-boolean, the condition may evaluate unexpectedly.
  - Parallel downstream nodes can fail independently; consider adding error handling if one notification fails but you still want logging.

---

### Block 4 — Automated Notifications

**Overview:** Sends customer email and internal Slack alerts according to fulfillment availability.

**Nodes involved:**
- Group Sticky 4 (documentation)
- Send Customer Confirmation1
- Notify Fulfillment Team1
- Send Backorder Notice1
- Backorder Alert1
- Merge Processing Paths1 (because it consolidates these notification branches)

#### Node: **Send Customer Confirmation1**
- **Type / role:** `Gmail` — emails the customer when order is confirmed.
- **Configuration choices:**
  - **To:** `{{ $json.customer.email }}`
  - **Subject:** `Order Confirmed`
  - **Message:** `Thank you!`
- **Connections:**
  - Input ← **Check Fulfillment Status1** (true path)
  - Output → **Merge Processing Paths1** (branch index 0)
- **Version requirements:** `typeVersion: 2.1`
- **Potential failures / edge cases:**
  - Gmail OAuth2 credential missing/expired; API scopes insufficient
  - Invalid/empty recipient email (missing billing email in Woo order)
  - Gmail sending limits / rate limiting

#### Node: **Notify Fulfillment Team1**
- **Type / role:** `Slack` — sends Slack message for new fulfillable order.
- **Configuration choices:**
  - **Text:** `New Order!`
  - Channel is not explicitly set in parameters shown; depends on Slack credential defaults or node configuration in UI.
- **Connections:**
  - Input ← **Check Fulfillment Status1** (true path)
  - Output → **Merge Processing Paths1** (branch index 0)
- **Version requirements:** `typeVersion: 2.2`
- **Potential failures / edge cases:**
  - Slack OAuth/token misconfiguration
  - Missing channel selection / not permitted to post
  - Message is minimal; may be insufficient operationally (consider including order number, items, totals)

#### Node: **Send Backorder Notice1**
- **Type / role:** `Gmail` — emails the customer when some items are on backorder.
- **Configuration choices:**
  - **To:** `{{ $json.customer.email }}`
  - **Subject:** `Backorder notice`
  - **Message:** `Some items are on backorder.`
- **Connections:**
  - Input ← **Check Fulfillment Status1** (false path)
  - Output → **Merge Processing Paths1** (branch index 1)
- **Version requirements:** `typeVersion: 2.1`
- **Potential failures / edge cases:** same as confirmation email (auth, invalid recipient, quotas)

#### Node: **Backorder Alert1**
- **Type / role:** `Slack` — alerts internal team about backorder.
- **Configuration choices:**
  - **Text:** `Backorder alert!`
- **Connections:**
  - Input ← **Check Fulfillment Status1** (false path)
  - Output → **Merge Processing Paths1** (branch index 1)
- **Version requirements:** `typeVersion: 2.2`
- **Potential failures / edge cases:** same as Slack node above

#### Node: **Merge Processing Paths1**
- **Type / role:** `Merge` (chooseBranch) — consolidates the “in stock” and “backorder” paths into a single flow for logging.
- **Configuration choices:**
  - **Mode:** Choose Branch (whichever branch produces data continues)
- **Connections:**
  - Input 1 ← **Send Customer Confirmation1** and **Notify Fulfillment Team1**
  - Input 2 ← **Send Backorder Notice1** and **Backorder Alert1**
  - Output → **Log Order to Sheets1**
- **Version requirements:** `typeVersion: 3`
- **Potential failures / edge cases:**
  - With parallel notifications, multiple nodes feed the same merge input; “chooseBranch” means the exact item that continues may depend on execution timing. This can make downstream logging inconsistent (you may log Slack output vs Gmail output rather than the unified order object).
  - If you need deterministic logging, add a separate “Set” node to carry the order object through, or merge by key with “Wait for Both” strategy.

---

### Block 5 — Logging & Response

**Overview:** Logs final status into Google Sheets and responds to the original webhook call.

**Nodes involved:**
- Group Sticky 5 (documentation)
- Log Order to Sheets1
- Respond to Webhook1

#### Node: **Log Order to Sheets1**
- **Type / role:** `Google Sheets` — appends a row for the processed order.
- **Configuration choices:**
  - **Operation:** Append
  - **Document ID / Sheet Name:** configured via UI resource locator fields but currently empty in JSON export (`value: ""`)
  - Columns mapping is not shown; in n8n, append typically requires specifying fields/columns in the node UI.
- **Connections:**
  - Input ← **Merge Processing Paths1**
  - Output → **Respond to Webhook1**
- **Version requirements:** `typeVersion: 4.5`
- **Potential failures / edge cases:**
  - Missing Google credentials or wrong scopes (Sheets API)
  - Spreadsheet ID / sheet name not set
  - Header mismatch: if append expects certain columns and the incoming JSON doesn’t match, rows may be incomplete or append may fail
  - Because upstream merge may pass Gmail/Slack response JSON, the data logged may not contain `orderId`, etc., unless you explicitly map from the original order object.

#### Node: **Respond to Webhook1**
- **Type / role:** `Respond to Webhook` — returns final HTTP response for webhook executions.
- **Configuration choices:**
  - **Respond with:** JSON
  - **Body:** `{{ {success: true} }}`
- **Connections:**
  - Input ← **Log Order to Sheets1**
- **Version requirements:** `typeVersion: 1.1`
- **Potential failures / edge cases:**
  - If the workflow is triggered by **Scheduled Order Sync1**, there is no webhook caller; the node may not be needed or may error depending on n8n execution context.
  - If any node errors before this point, webhook requests may time out.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Main Sticky Note | Sticky Note | Documentation / setup notes | — | — | ## How it works … Sheet Prep: Specify the Spreadsheet ID and Sheet Name where order data should be logged. |
| Group Sticky 1 | Sticky Note | Block label: Order Intake | — | — | ### 1. Order Intake Capture order data via Webhook or scheduled sync. Converts raw data into a clean JSON format for processing. |
| Group Sticky 2 | Sticky Note | Block label: Inventory Check | — | — | ### 2. Inventory Check Splits the order into individual items to check stock levels. Determines if the entire order can be fulfilled immediately. |
| Group Sticky 3 | Sticky Note | Block label: Processing Logic | — | — | ### 3. Processing Logic Calculates shipping costs and determines fulfillment priority. Branches the workflow based on stock availability. |
| Group Sticky 4 | Sticky Note | Block label: Automated Notifications | — | — | ### 4. Automated Notifications Sends confirmation or backorder emails to the customer and triggers Slack alerts for the fulfillment/inventory teams. |
| Group Sticky 5 | Sticky Note | Block label: Logging & Response | — | — | ### 5. Logging & Response Logs the final order status to Google Sheets and returns a success response to the triggering Webhook. |
| WooCommerce Order Webhook1 | Webhook | Real-time order intake | — | Merge Triggers1 | ### 1. Order Intake … |
| Scheduled Order Sync1 | Schedule Trigger | Periodic trigger (hourly) | — | Merge Triggers1 | ### 1. Order Intake … |
| Merge Triggers1 | Merge | Unify trigger paths | WooCommerce Order Webhook1; Scheduled Order Sync1 | Extract Order Data1 | ### 1. Order Intake … |
| Extract Order Data1 | Code | Normalize Woo order payload | Merge Triggers1 | Split Order Items1 | ### 1. Order Intake … |
| Split Order Items1 | Split Out | Create one execution item per line item | Extract Order Data1 | Check Inventory1 | ### 2. Inventory Check … |
| Check Inventory1 | Code | Inventory lookup (simulated) | Split Order Items1 | Aggregate Inventory Results1 | ### 2. Inventory Check … |
| Aggregate Inventory Results1 | Aggregate | Combine per-item inventory checks | Check Inventory1 | Process Order Logic1 | ### 2. Inventory Check … |
| Process Order Logic1 | Code | Decide fulfillment + shipping | Aggregate Inventory Results1 | Check Fulfillment Status1 | ### 3. Processing Logic … |
| Check Fulfillment Status1 | IF | Branch by in-stock vs backorder | Process Order Logic1 | Send Customer Confirmation1; Notify Fulfillment Team1; Send Backorder Notice1; Backorder Alert1 | ### 3. Processing Logic … |
| Send Customer Confirmation1 | Gmail | Customer email (confirmed) | Check Fulfillment Status1 (true) | Merge Processing Paths1 | ### 4. Automated Notifications … |
| Notify Fulfillment Team1 | Slack | Slack notification (fulfillment) | Check Fulfillment Status1 (true) | Merge Processing Paths1 | ### 4. Automated Notifications … |
| Send Backorder Notice1 | Gmail | Customer email (backorder) | Check Fulfillment Status1 (false) | Merge Processing Paths1 | ### 4. Automated Notifications … |
| Backorder Alert1 | Slack | Slack alert (backorder) | Check Fulfillment Status1 (false) | Merge Processing Paths1 | ### 4. Automated Notifications … |
| Merge Processing Paths1 | Merge | Unify notification branches | Send Customer Confirmation1; Notify Fulfillment Team1; Send Backorder Notice1; Backorder Alert1 | Log Order to Sheets1 | ### 5. Logging & Response … |
| Log Order to Sheets1 | Google Sheets | Append order log row | Merge Processing Paths1 | Respond to Webhook1 | ### 5. Logging & Response … |
| Respond to Webhook1 | Respond to Webhook | Return HTTP success | Log Order to Sheets1 | — | ### 5. Logging & Response … |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add Webhook node**
   - Name: `WooCommerce Order Webhook1`
   - Method: **POST**
   - Path: `woo-new-order`
   - Response: **Using “Respond to Webhook” node**
   - In WooCommerce, configure a webhook pointing to this URL (event like “Order created/updated” as desired).

3. **Add Schedule Trigger node**
   - Name: `Scheduled Order Sync1`
   - Interval: **Every 1 hour**
   - Note: to make this meaningful, add later a WooCommerce “Get Many Orders” node after this trigger (not present in current design).

4. **Add Merge node**
   - Name: `Merge Triggers1`
   - Mode: **Choose Branch**
   - Connect:
     - Webhook → Merge input 1
     - Schedule Trigger → Merge input 2

5. **Add Code node**
   - Name: `Extract Order Data1`
   - Paste logic to normalize payload (build `orderId`, `orderNumber`, `items[]`, `customer`, `shipping`, `total`).
   - Connect: Merge → Code

6. **Add Split Out node**
   - Name: `Split Order Items1`
   - Field to split out: `items`
   - Connect: Extract Order Data → Split Out

7. **Add Code node**
   - Name: `Check Inventory1`
   - Start with stub that adds `isInStock` and `stockStatus`, then replace with your real inventory integration (HTTP Request / DB / ERP node).
   - Connect: Split Out → Check Inventory

8. **Add Aggregate node**
   - Name: `Aggregate Inventory Results1`
   - Operation/Mode: **Aggregate all item data** (so results appear under a `data` array)
   - Connect: Check Inventory → Aggregate

9. **Add Code node**
   - Name: `Process Order Logic1`
   - Compute:
     - `inventory.allInStock` (currently hard-coded true)
     - `fulfillment.status/priority`
     - `shipping.selectedRate`
   - Reference normalized order object from `Extract Order Data1` (or pass it forward explicitly).
   - Connect: Aggregate → Process Order Logic

10. **Add IF node**
    - Name: `Check Fulfillment Status1`
    - Condition: Boolean `{{$json.inventory.allInStock}}` equals `true`
    - Connect: Process Order Logic → IF

11. **Add Gmail nodes (2)**
    - Node A name: `Send Customer Confirmation1`
      - To: `{{$json.customer.email}}`
      - Subject: `Order Confirmed`
      - Message: `Thank you!`
    - Node B name: `Send Backorder Notice1`
      - To: `{{$json.customer.email}}`
      - Subject: `Backorder notice`
      - Message: `Some items are on backorder.`
    - **Credentials:** configure Gmail OAuth2 in n8n and select it in both nodes.
    - Connect:
      - IF true → Confirmation
      - IF false → Backorder Notice

12. **Add Slack nodes (2)**
    - Node A name: `Notify Fulfillment Team1` with text `New Order!`
    - Node B name: `Backorder Alert1` with text `Backorder alert!`
    - **Credentials:** configure Slack OAuth2/token in n8n; set channel as needed.
    - Connect:
      - IF true → Notify Fulfillment Team
      - IF false → Backorder Alert

13. **Add Merge node**
    - Name: `Merge Processing Paths1`
    - Mode: **Choose Branch**
    - Connect outputs of the four notification nodes into the merge inputs (matching your preferred wiring).
    - Note: for deterministic logging, consider passing the order object to logging separately rather than relying on whichever notification finishes first.

14. **Add Google Sheets node**
    - Name: `Log Order to Sheets1`
    - Operation: **Append**
    - Set:
      - **Spreadsheet (Document ID)**: select your spreadsheet
      - **Sheet name**: select the target sheet/tab
    - Configure column mappings to include order identifiers (e.g., `orderId`, `orderNumber`, `total`, `fulfillment.status`, `inventory.allInStock`, timestamp, etc.).
    - **Credentials:** configure Google Sheets OAuth2 in n8n and select it.
    - Connect: Merge Processing Paths → Google Sheets

15. **Add Respond to Webhook node**
    - Name: `Respond to Webhook1`
    - Respond with: **JSON**
    - Body: `{"success": true}` (via expression or static)
    - Connect: Google Sheets → Respond to Webhook

16. **Activate workflow** after testing:
    - Test webhook with a sample WooCommerce payload.
    - Confirm emails and Slack messages.
    - Confirm a new row is appended to Google Sheets.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This pipeline automatically processes orders from WooCommerce. It automates inventory checks, shipping cost calculations, customer/team notifications based on order status, and logs all data to Google Sheets. | Main Sticky Note |
| Setup steps: configure WooCommerce webhook URL; implement real inventory integration in “Check Inventory”; set up Gmail/Slack/Google Sheets credentials; set Spreadsheet ID and Sheet Name for logging. | Main Sticky Note |